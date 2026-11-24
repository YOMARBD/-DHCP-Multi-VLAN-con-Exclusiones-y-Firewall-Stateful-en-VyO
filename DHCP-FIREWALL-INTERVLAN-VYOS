# 🛡️ Laboratorio 02: DHCP Multi-VLAN con Exclusiones y Firewall Stateful en VyOS

Implementación de una infraestructura de red segmentada en **GNS3** utilizando un router **VyOS 1.4+** (*Router-on-a-Stick*), subnetting **`/26`**, asignación dinámica mediante **Servidor DHCP** (con exclusión de direcciones IP fijas) y políticas de **Firewall Stateful (`forward filter`)** para el aislamiento asimétrico de departamentos.

---

## 1. Topología de Red

*()*

---

## 2. Esquema de Direccionamiento IP y Subnetting (`/26`)

Partiendo de la red base **`10.10.0.0/24`**, se prestaron **2 bits** (\(2^2 = 4\text{ subredes}\)) obteniendo la máscara **`/26` (`255.255.255.192`)** con bloques de 64 direcciones (62 hosts utilizables por subred):

| Departamento | VLAN | ID de Red | Puerta de Enlace (VyOS) | Rango DHCP Asignado | Broadcast | Política de Seguridad |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Docentes** | 10 | `10.10.0.0/26` | `10.10.0.1` (`eth0.10`) | `.10` – `.50` | `10.10.0.63` | Acceso total hacia Alumnos |
| **Alumnos 1** | 20 | `10.10.0.64/26` | `10.10.0.65` (`eth0.20`) | `.75` – `.120` | `10.10.0.127` | Bloqueado hacia Docentes |
| **Alumnos 2** | 30 | `10.10.0.128/26` | `10.10.0.129` (`eth0.30`) | `.140`–`.149` y `.151`–`.180` *(Excluye `.150`)* | `10.10.0.191` | Bloqueado hacia Docentes |

---

## 3. Configuración del Router (`R1-FACULTAD` - VyOS 1.4+)

### A. Subinterfaces VLAN (802.1Q)
```bash
configure

set interfaces ethernet eth0 vif 10 address 10.10.0.1/26
set interfaces ethernet eth0 vif 10 description "VLAN_10_DOCENTES"

set interfaces ethernet eth0 vif 20 address 10.10.0.65/26
set interfaces ethernet eth0 vif 20 description "VLAN_20_ALUMNOS1"

set interfaces ethernet eth0 vif 30 address 10.10.0.129/26
set interfaces ethernet eth0 vif 30 description "VLAN_30_ALUMNOS2"
```

### B. Servidor DHCP Multi-VLAN con Exclusión de Rango
En la **VLAN 30** se divide el reparto en `range 0` y `range 1` para reservar la IP `10.10.0.150` para un servidor estático:

```bash
# Pool VLAN 10 (Docentes)
set service dhcp-server shared-network-name VLAN10_POOL subnet 10.10.0.0/26 subnet-id 10
set service dhcp-server shared-network-name VLAN10_POOL subnet 10.10.0.0/26 option default-router 10.10.0.1
set service dhcp-server shared-network-name VLAN10_POOL subnet 10.10.0.0/26 range 0 start 10.10.0.10
set service dhcp-server shared-network-name VLAN10_POOL subnet 10.10.0.0/26 range 0 stop 10.10.0.50

# Pool VLAN 20 (Alumnos 1)
set service dhcp-server shared-network-name VLAN20_POOL subnet 10.10.0.64/26 subnet-id 20
set service dhcp-server shared-network-name VLAN20_POOL subnet 10.10.0.64/26 option default-router 10.10.0.65
set service dhcp-server shared-network-name VLAN20_POOL subnet 10.10.0.64/26 range 0 start 10.10.0.75
set service dhcp-server shared-network-name VLAN20_POOL subnet 10.10.0.64/26 range 0 stop 10.10.0.120

# Pool VLAN 30 (Alumnos 2 - Exclusión de .150)
set service dhcp-server shared-network-name VLAN30_POOL subnet 10.10.0.128/26 subnet-id 30
set service dhcp-server shared-network-name VLAN30_POOL subnet 10.10.0.128/26 option default-router 10.10.0.129
set service dhcp-server shared-network-name VLAN30_POOL subnet 10.10.0.128/26 range 0 start 10.10.0.140
set service dhcp-server shared-network-name VLAN30_POOL subnet 10.10.0.128/26 range 0 stop 10.10.0.149
set service dhcp-server shared-network-name VLAN30_POOL subnet 10.10.0.128/26 range 1 start 10.10.0.151
set service dhcp-server shared-network-name VLAN30_POOL subnet 10.10.0.128/26 range 1 stop 10.10.0.180
```

### C. Firewall Stateful (`ipv4 forward filter`)
Se implementa aislamiento asimétrico permitiendo las respuestas de conexiones iniciadas por Docentes (`established` / `related`) y descartando (`drop`) cualquier tráfico nuevo originado desde las VLANs de Alumnos hacia Docentes:

```bash
set firewall ipv4 forward filter default-action accept

# Regla 10: Permitir respuestas de conexiones iniciadas por Docentes
set firewall ipv4 forward filter rule 10 action accept
set firewall ipv4 forward filter rule 10 state established
set firewall ipv4 forward filter rule 10 state related

# Regla 20: Bloquear tráfico nuevo desde Alumnos 1 hacia Docentes
set firewall ipv4 forward filter rule 20 action drop
set firewall ipv4 forward filter rule 20 source address 10.10.0.64/26
set firewall ipv4 forward filter rule 20 destination address 10.10.0.0/26
set firewall ipv4 forward filter rule 20 description "Bloquear_Alumnos1_a_Docentes"

# Regla 30: Bloquear tráfico nuevo desde Alumnos 2 hacia Docentes
set firewall ipv4 forward filter rule 30 action drop
set firewall ipv4 forward filter rule 30 source address 10.10.0.128/26
set firewall ipv4 forward filter rule 30 destination address 10.10.0.0/26
set firewall ipv4 forward filter rule 30 description "Bloquear_Alumnos2_a_Docentes"

commit
save
exit
```

---

## 4. Configuración del Switch y Clientes (`VPCS`)

### Switch (`SW-LAB`)
* **Puerto hacia `R1-FACULTAD`:** Type `dot1q` | VLAN `1` (Troncal)
* **Puerto hacia `PC-DOCENTES`:** Type `access` | VLAN `10`
* **Puerto hacia `PC-ALUMNOS-1`:** Type `access` | VLAN `20`
* **Puerto hacia `PC-ALUMNOS-2`:** Type `access` | VLAN `30`

### Clientes (`VPCS`)
```bash
ip dhcp
save
```

---

## 5. Verificación y Pruebas de Seguridad

### Tabla de Concesiones DHCP (`show dhcp server leases`)
*(Pega aquí con `Ctrl + V` tu captura de `show dhcp server leases` mostrando las 3 IPs activas)*

### Pruebas Cruzadas de Conectividad y Firewall
* **Desde `PC-DOCENTES` (`10.10.0.10`) hacia Alumnos:** Respuesta exitosa con **`ttl=63`** (permitida por `rule 10`).
* **Desde `PC-ALUMNOS-1` y `PC-ALUMNOS-2` hacia Docentes (`10.10.0.10`):** **`timeout`** (bloqueada silenciosamente por `rule 20` y `rule 30`).
* **Entre `PC-ALUMNOS-1` y `PC-ALUMNOS-2`:** Respuesta exitosa con **`ttl=63`** (permitida por `default-action accept`).

*(Pega aquí con `Ctrl + V` tus capturas de las consolas de PC1, PC2 y PC3)*
