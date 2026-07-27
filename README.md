# 🏬 Mall Anillo 5 — Diseño y Segmentación de Red con VLANs

> ⚠️ **Proyecto en desarrollo activo.**
> **Fase actual completada:** diseño VLSM, configuración de VLANs, trunking, routing inter-VLAN (router-on-a-stick) y ACLs de seguridad entre VLANs — todo verificado con pruebas reales de conectividad.
> **Próximos pasos:** VLAN de invitados (WiFi), servidor DHCP con DHCP Relay, documentación de hardening.

## 📖 Contexto del proyecto

Este proyecto simula el diseño de red de un centro comercial ficticio, **Mall Anillo 5**, compuesto por 310 locales comerciales distribuidos en 3 plantas más una planta de patio de comidas. El objetivo es aplicar de forma práctica conceptos de segmentación de red (VLANs), subnetting con VLSM, trunking 802.1Q y routing inter-VLAN mediante un router-on-a-stick, usando Cisco Packet Tracer.

El proyecto nace como ejercicio de refuerzo personal en la asignatura de Redes, con el objetivo de consolidar en un caso realista los conceptos que resultan más abstractos al estudiarlos solo en teoría.

## 🎯 Objetivos técnicos

- Diseñar un esquema de direccionamiento IP eficiente mediante **VLSM** para 8 segmentos de red con necesidades de hosts muy distintas (desde 4 hasta 2000+ direcciones).
- Segmentar la red en **VLANs** por planta y por función (comercial, administración, seguridad, servidores, invitados).
- Configurar **enlaces trunk (802.1Q)** entre switches de zona y el switch núcleo.
- Configurar un **router-on-a-stick** con sub-interfaces para permitir la comunicación entre VLANs.
- Verificar la conectividad con pruebas reales (ping, análisis de TTL) documentando evidencias.

## 🗺️ Topología de red

Arquitectura jerárquica en árbol: un switch núcleo centraliza el tráfico de 4 switches de zona (uno por planta/área) y da salida a un router y un servidor.

```
                         [Router0]
                             |
                        [Server0]
                             |
                       [Switch0 - Core]
                 /        |         |        \
           [Sw-P1]   [Sw-P2]   [Sw-P3]  [Sw-ADMIN-PATIO-CCTVS]
              |          |         |              |
          PC0, PC1   PC2, PC3  PC4, PC5      PC6, PC7
```

![Topología general de la red](01-topologia-general.png)

## 🧮 Diseño de direccionamiento (VLSM)

Bloque de red de partida: **192.168.0.0/19** (8192 direcciones), calculado a partir de la necesidad total estimada de ~4635 hosts, con margen de crecimiento.

| VLAN | ID | Zona | Hosts estimados | Máscara | Red | Rango usable | Broadcast |
|---|---|---|---|---|---|---|---|
| WIFI | — | Invitados | 2000 | /21 | 192.168.0.0 | .0.1 – .7.254 | 192.168.7.255 |
| P1 | 10 | Planta 1 | 736 | /22 | 192.168.8.0 | .8.1 – .11.254 | 192.168.11.255 |
| P2 | 20 | Planta 2 | 736 | /22 | 192.168.12.0 | .12.1 – .15.254 | 192.168.15.255 |
| P3 | 30 | Planta 3 | 728 | /22 | 192.168.16.0 | .16.1 – .19.254 | 192.168.19.255 |
| PATIO | 50 | Patio de comidas | 280 | /23 | 192.168.20.0 | .20.1 – .21.254 | 192.168.21.255 |
| ADMIN | 40 | Administración | 120 | /25 | 192.168.22.0 | .22.1 – .22.126 | 192.168.22.127 |
| CCTV | 60 | Cámaras/seguridad | 32 | /27 | 192.168.22.128 | .22.129 – .22.158 | 192.168.22.159 |
| SERVIDORES | 70 | Servidores internos | 3-4 | /30 | 192.168.22.160 | .22.161 – .22.162 | 192.168.22.163 |

Todos los cálculos fueron realizados manualmente y verificados posteriormente con `ipcalc` en Linux.

![Verificación VLSM con ipcalc](02-ipcalc-verificacion.png)

## ⚙️ Configuración implementada

### Switches de zona (P1, P2, P3, ADMIN-PATIO-CCTVS)
- Creación de VLANs correspondientes a cada zona.
- Puertos de acceso (`switchport mode access`) para los hosts finales.
- Puerto de subida al núcleo configurado como trunk (`switchport mode trunk`).

### Switch núcleo (Switch0)
- Las 7 VLANs (10, 20, 30, 40, 50, 60, 70) creadas en el switch núcleo.
- 4 puertos trunk hacia los switches de zona.
- 1 puerto trunk hacia el router (para el router-on-a-stick).
- 1 puerto de acceso (VLAN 70) hacia el servidor.

### Router0 — Router-on-a-Stick
Sub-interfaz dedicada por cada VLAN sobre una única interfaz física (`GigabitEthernet0/0`), usando encapsulación 802.1Q:

| Sub-interfaz | VLAN | IP Gateway |
|---|---|---|
| Gi0/0.10 | P1 | 192.168.8.1 |
| Gi0/0.20 | P2 | 192.168.12.1 |
| Gi0/0.30 | P3 | 192.168.16.1 |
| Gi0/0.40 | ADMIN | 192.168.22.1 |
| Gi0/0.50 | PATIO | 192.168.20.1 |
| Gi0/0.60 | CCTV | 192.168.22.129 |
| Gi0/0.70 | SERVIDORES | 192.168.22.161 |

![Sub-interfaces del router activas](03-router-show-ip-interface-brief.png)

## ✅ Verificación y pruebas realizadas

| Prueba | Resultado | Qué demuestra |
|---|---|---|
| Ping PC0 → Gateway (192.168.8.1) | 4/4 recibidos | Conectividad host → router dentro de la misma VLAN |
| Ping PC0 → PC1 (misma VLAN) | 4/4 recibidos | Comunicación intra-VLAN a través del switch de acceso |
| Ping PC0 (VLAN 10) → PC2 (VLAN 20) | 3/4 recibidos, TTL=127 | Routing inter-VLAN funcional (el TTL decrementado en 1 confirma el salto por el router) |

**Ping al gateway:**
![Ping PC0 a gateway](04-ping-mismo-gateway.png)

**Ping dentro de la misma VLAN:**
![Ping PC0 a PC1](05-ping-misma-vlan.png)

**Ping entre VLANs distintas (routing inter-VLAN):**
![Ping PC0 a PC2 entre VLANs](06-ping-entre-vlans.png)

## 🔒 Seguridad — ACLs entre VLANs

Una vez verificado que el routing inter-VLAN funcionaba, se aplicaron **Listas de Control de Acceso (ACL extendidas)** en el Router0 para restringir la comunicación entre VLANs según reglas de negocio realistas, en lugar de dejar toda la red completamente abierta.

### Reglas de negocio definidas

| Origen | Destino | ¿Permitido? | Motivo |
|---|---|---|---|
| P1, P2, P3 (locales) | ADMIN, CCTV, SERVIDORES | ❌ No | Cada tienda es una empresa independiente, sin relación con la gestión interna del mall |
| P1, P2, P3 (locales) | entre sí | ❌ No | Las tiendas no tienen motivo para verse entre ellas |
| WIFI (invitados) | Todo lo interno (P1, P2, P3, ADMIN, CCTV, SERVIDORES) | ❌ No | Los visitantes solo deben tener salida a internet |
| ADMIN | CCTV, SERVIDORES | ✅ Sí | El personal de gestión del mall necesita supervisar cámaras y administrar servidores |
| Cualquiera | Internet | ✅ Sí | Regla final `permit ip any any` |

### Cálculo de wildcard masks

Las ACLs de Cisco usan wildcard masks en lugar de máscaras de subred estándar (funcionan de forma invertida: 0 = debe coincidir, 1 = no importa). Se calcularon restando cada máscara a 255.255.255.255:

| VLAN | Red | Máscara | Wildcard mask |
|---|---|---|---|
| P1 | 192.168.8.0 | 255.255.252.0 | 0.0.3.255 |
| P2 | 192.168.12.0 | 255.255.252.0 | 0.0.3.255 |
| P3 | 192.168.16.0 | 255.255.252.0 | 0.0.3.255 |
| ADMIN | 192.168.22.0 | 255.255.255.128 | 0.0.0.127 |
| CCTV | 192.168.22.128 | 255.255.255.224 | 0.0.0.31 |
| SERVIDORES | 192.168.22.160 | 255.255.255.252 | 0.0.0.3 |
| WIFI | 192.168.0.0 | 255.255.248.0 | 0.0.7.255 |

### Implementación

Se creó una única ACL extendida (**access-list 100**) con 21 reglas de denegación específicas más una regla final de permiso general, aplicada en modo **entrante (`in`)** sobre las sub-interfaces de origen con restricciones (Gi0/0.10, Gi0/0.20, Gi0/0.30):

```
access-list 100 deny ip 192.168.8.0 0.0.3.255 192.168.22.0 0.0.0.127
access-list 100 deny ip 192.168.8.0 0.0.3.255 192.168.22.128 0.0.0.31
access-list 100 deny ip 192.168.8.0 0.0.3.255 192.168.22.160 0.0.0.3
[...]
access-list 100 permit ip any any
```

```
interface GigabitEthernet0/0.10
 ip access-group 100 in
```

![Listado completo de la ACL 100](07-acl-show-access-list.png)

![ACL aplicada en la sub-interfaz de P1](08-acl-verificacion-interfaz.png)

### Verificación con pruebas reales

| Prueba | Resultado | Qué demuestra |
|---|---|---|
| Ping PC0 (P1) → 192.168.22.2 (ADMIN) | 100% perdido, "Destination host unreachable" | La ACL bloquea correctamente el tráfico hacia VLANs restringidas |
| Ping PC0 (P1) → 192.168.8.1 (su propio gateway) | 4/4 recibidos | La ACL no afecta al tráfico legítimo permitido |

**Ping bloqueado (P1 → ADMIN):**
![Ping bloqueado por la ACL](09-ping-bloqueado-por-acl.png)

**Ping permitido (P1 → su gateway):**
![Ping permitido al gateway](10-ping-permitido-gateway.png)

## 🛠️ Herramientas utilizadas

- **Cisco Packet Tracer** — simulación de la topología y configuración de dispositivos.
- **ipcalc** (Linux) — verificación de cálculos VLSM.
- CLI de IOS (switches y router) para toda la configuración.

## 📌 Próximos pasos (roadmap)

- [x] Configurar **ACLs** para restringir tráfico entre VLANs según reglas de negocio (ej. Invitados sin acceso a Servidores/CCTV, Locales sin acceso a Administración).
- [ ] Montar la **VLAN de invitados (WiFi)** en switches (actualmente solo diseñada en el VLSM) y aplicarle su ACL correspondiente.
- [ ] Implementar **servidor DHCP** con **DHCP Relay** para reparto automático de IPs entre VLANs.
- [ ] Añadir documentación de hardening y buenas prácticas de seguridad en switches (port-security, DTP desactivado, etc.).
- [ ] Migrar la topología a **GNS3** para integración con máquinas reales (ej. Wazuh SIEM) y tráfico de ataque real.

## 👤 Autor

Alberto — Técnico en Administración de Sistemas Informáticos en Red (ASIR).
[GitHub](https://github.com/alberto-seco)
