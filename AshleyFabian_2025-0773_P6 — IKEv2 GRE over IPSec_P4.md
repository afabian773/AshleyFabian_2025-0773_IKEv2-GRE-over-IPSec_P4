# VPN IPSec IKEv2 Site-to-Site con Túnel GRE (GRE over IPSec)

**Nombre:** Ashley Fabian Ortiz  
**Matrícula:** 2025-0773  
**Práctica:** P4  

---

## Objetivo

Configurar una VPN Site-to-Site IPSec IKEv2 utilizando un túnel **GRE (Generic Routing Encapsulation)** protegido mediante IPSec en modo Transporte (GRE over IPSec), entre dos routers Cisco CSR1000v, permitiendo la comunicación cifrada entre dos redes LAN a través de un router ISP. Esta práctica combina la arquitectura GRE over IPSec de P3 con el protocolo de negociación IKEv2 de P4/P5, representando la implementación más moderna de esta topología.

La diferencia fundamental respecto a P3 (IKEv1 GRE over IPSec) es la sustitución de `crypto isakmp policy/key` por el conjunto `crypto ikev2 proposal/policy/keyring/profile`, y la adición de `set ikev2-profile` en el crypto map. La arquitectura GRE + IPSec Transport permanece idéntica.

### Comparación P3 (IKEv1) vs P6 (IKEv2) — GRE over IPSec

| Característica            | P3 (IKEv1 GRE)                  | P6 (IKEv2 GRE)                        |
|---------------------------|----------------------------------|----------------------------------------|
| Negociación fase 1        | `crypto isakmp policy`           | `crypto ikev2 proposal/policy/profile` |
| Llave precompartida       | `crypto isakmp key ... address`  | `crypto ikev2 keyring` (bloque peer)   |
| Referencia en crypto map  | Implícita (IKEv1 por defecto)    | `set ikev2-profile PROF-P6`            |
| Mensajes de negociación   | 6-9 (Main Mode)                  | 4 (IKE_SA_INIT + IKE_AUTH)             |
| Comando de verificación   | `show crypto isakmp sa`          | `show crypto ikev2 sa`                 |
| Estado SA activa          | `QM_IDLE`                        | `READY`                                |
| Túnel GRE                 | Igual                            | Igual                                  |
| Modo IPSec                | Transport                        | Transport                              |
| ACL (tráfico interesante) | `permit gre host X host Y`       | `permit gre host X host Y`             |

---

## Topología

```
[PC1]--[SW-A]--[R1]--------[ISP]--------[R2]--[SW-B]--[PC2]
        LAN-A   Gi3    Gi1       Gi2    Gi2    LAN-B
              25.7.73.169  25.7.73.161  25.7.73.165  25.7.73.166  25.7.73.177

                       Tunnel0 (GRE)         Tunnel0 (GRE)
                    25.7.73.185 <----------> 25.7.73.186
                      Protegido por IPSec IKEv2 (modo Transport)
```

---

## Direccionamiento IP

| Dispositivo | Interfaz         | Dirección IP  | Máscara         | Descripción        |
|-------------|------------------|---------------|-----------------|--------------------|
| R1          | GigabitEthernet1 | 25.7.73.161   | 255.255.255.252 | WAN R1 → ISP       |
| ISP         | GigabitEthernet1 | 25.7.73.162   | 255.255.255.252 | WAN ISP → R1       |
| ISP         | GigabitEthernet2 | 25.7.73.165   | 255.255.255.252 | WAN ISP → R2       |
| R2          | GigabitEthernet2 | 25.7.73.166   | 255.255.255.252 | WAN R2 → ISP       |
| R1          | GigabitEthernet3 | 25.7.73.169   | 255.255.255.248 | Gateway LAN-A      |
| PC1         | eth0             | 25.7.73.170   | 255.255.255.248 | Host LAN-A         |
| R2          | GigabitEthernet3 | 25.7.73.177   | 255.255.255.248 | Gateway LAN-B      |
| PC2         | eth0             | 25.7.73.178   | 255.255.255.248 | Host LAN-B         |
| R1          | Tunnel0 (GRE)    | 25.7.73.185   | 255.255.255.252 | Extremo GRE R1     |
| R2          | Tunnel0 (GRE)    | 25.7.73.186   | 255.255.255.252 | Extremo GRE R2     |

### Subredes utilizadas (bloque 25.7.73.160/27)

| Subred           | Rango utilizable            | Uso            |
|-------------------|-----------------------------|----------------|
| 25.7.73.160/30    | 25.7.73.161 – 25.7.73.162   | WAN R1 – ISP   |
| 25.7.73.164/30    | 25.7.73.165 – 25.7.73.166   | WAN ISP – R2   |
| 25.7.73.168/29    | 25.7.73.169 – 25.7.73.174   | LAN-A          |
| 25.7.73.176/29    | 25.7.73.177 – 25.7.73.182   | LAN-B          |
| 25.7.73.184/30    | 25.7.73.185 – 25.7.73.186   | Túnel GRE      |

---

## Parámetros de configuración

### Túnel GRE

| Parámetro           | R1              | R2              |
|---------------------|-----------------|-----------------|
| IP Tunnel           | 25.7.73.185/30  | 25.7.73.186/30  |
| Tunnel source       | Gi1             | Gi2             |
| Tunnel destination  | 25.7.73.166     | 25.7.73.161     |
| Tunnel mode         | gre ip          | gre ip          |

### IKEv2 Proposal

| Parámetro  | Valor               |
|------------|---------------------|
| Cifrado    | AES-CBC 256         |
| Integridad | SHA-256             |
| Grupo DH   | Group 14 (2048-bit) |

### IKEv2 Keyring

| Parámetro       | R1                | R2                |
|-----------------|-------------------|-------------------|
| Peer remoto     | R2 (25.7.73.166)  | R1 (25.7.73.161)  |
| Pre-shared key  | Cisco123!         | Cisco123!         |

### IKEv2 Profile

| Parámetro             | R1                | R2                |
|-----------------------|-------------------|-------------------|
| Match identity remote | 25.7.73.166/32    | 25.7.73.161/32    |
| Autenticación local   | pre-share         | pre-share         |
| Autenticación remota  | pre-share         | pre-share         |

### Fase 2 — IPSec Transform Set

| Parámetro      | Valor             |
|----------------|-------------------|
| Cifrado ESP    | AES 256           |
| Integridad ESP | SHA-256 HMAC      |
| Modo           | **Transport**     |

### Tráfico interesante (ACL) — cifra protocolo GRE

| Router | Regla                                         |
|--------|-----------------------------------------------|
| R1     | `permit gre host 25.7.73.161 host 25.7.73.166` |
| R2     | `permit gre host 25.7.73.166 host 25.7.73.161` |

---

## Scripts de configuración

### ISP

```
enable
configure terminal
hostname ISP

interface GigabitEthernet1
 ip address 25.7.73.162 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet2
 ip address 25.7.73.165 255.255.255.252
 no shutdown
 exit

end
write memory
```

### R1

```
enable
configure terminal
hostname R1

interface GigabitEthernet1
 ip address 25.7.73.161 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.169 255.255.255.248
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 25.7.73.162

! --- Túnel GRE ---
interface Tunnel0
 ip address 25.7.73.185 255.255.255.252
 tunnel source GigabitEthernet1
 tunnel destination 25.7.73.166
 tunnel mode gre ip
 no shutdown
 exit

! --- Ruta hacia LAN-B por el túnel GRE ---
ip route 25.7.73.176 255.255.255.248 25.7.73.186

! --- IKEv2 Proposal ---
crypto ikev2 proposal PROP-P6
 encryption aes-cbc-256
 integrity sha256
 group 14
 exit

! --- IKEv2 Policy ---
crypto ikev2 policy POL-P6
 proposal PROP-P6
 exit

! --- Keyring ---
crypto ikev2 keyring KEY-P6
 peer R2
  address 25.7.73.166
  pre-shared-key Cisco123!
  exit
 exit

! --- IKEv2 Profile ---
crypto ikev2 profile PROF-P6
 match identity remote address 25.7.73.166 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KEY-P6
 exit

! --- Transform Set en modo Transport ---
crypto ipsec transform-set TS-P6 esp-aes 256 esp-sha256-hmac
 mode transport
 exit

! --- ACL: cifra protocolo GRE (47) entre IPs WAN ---
access-list 101 permit gre host 25.7.73.161 host 25.7.73.166

! --- Crypto Map referenciando IKEv2 profile ---
crypto map CMAP-P6 10 ipsec-isakmp
 set peer 25.7.73.166
 set transform-set TS-P6
 set ikev2-profile PROF-P6
 match address 101
 exit

interface GigabitEthernet1
 crypto map CMAP-P6
 exit

end
write memory
```

### R2

```
enable
configure terminal
hostname R2

interface GigabitEthernet2
 ip address 25.7.73.166 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.177 255.255.255.248
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 25.7.73.165

! --- Túnel GRE ---
interface Tunnel0
 ip address 25.7.73.186 255.255.255.252
 tunnel source GigabitEthernet2
 tunnel destination 25.7.73.161
 tunnel mode gre ip
 no shutdown
 exit

! --- Ruta hacia LAN-A por el túnel GRE ---
ip route 25.7.73.168 255.255.255.248 25.7.73.185

! --- IKEv2 Proposal ---
crypto ikev2 proposal PROP-P6
 encryption aes-cbc-256
 integrity sha256
 group 14
 exit

! --- IKEv2 Policy ---
crypto ikev2 policy POL-P6
 proposal PROP-P6
 exit

! --- Keyring ---
crypto ikev2 keyring KEY-P6
 peer R1
  address 25.7.73.161
  pre-shared-key Cisco123!
  exit
 exit

! --- IKEv2 Profile ---
crypto ikev2 profile PROF-P6
 match identity remote address 25.7.73.161 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KEY-P6
 exit

! --- Transform Set en modo Transport ---
crypto ipsec transform-set TS-P6 esp-aes 256 esp-sha256-hmac
 mode transport
 exit

! --- ACL: cifra protocolo GRE (47) entre IPs WAN ---
access-list 101 permit gre host 25.7.73.166 host 25.7.73.161

! --- Crypto Map referenciando IKEv2 profile ---
crypto map CMAP-P6 10 ipsec-isakmp
 set peer 25.7.73.161
 set transform-set TS-P6
 set ikev2-profile PROF-P6
 match address 101
 exit

interface GigabitEthernet2
 crypto map CMAP-P6
 exit

end
write memory
```

### SW-A y SW-B (IOSvL2)

```
enable
configure terminal

interface GigabitEthernet0/0
 switchport mode access
 switchport access vlan 1
 no shutdown
 exit

interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 1
 no shutdown
 exit

end
write memory
```

### PC1 (VPCS)

```
ip 25.7.73.170 255.255.255.248 25.7.73.169
save
```

### PC2 (VPCS)

```
ip 25.7.73.178 255.255.255.248 25.7.73.177
save
```

---

## Verificación y pruebas

> Comandos ejecutados en R1 y R2. El ISP no participa de la VPN; los PCs solo ejecutan `ping`.

### Comandos de verificación

```
show interface Tunnel0
show crypto ikev2 sa
show crypto ipsec sa
show ip route
```

### Resultados obtenidos — R1

**show interface Tunnel0:**
```
Tunnel0 is up, line protocol is up
  Internet address is 25.7.73.185/30
  Tunnel protocol/transport GRE/IP
  Tunnel source 25.7.73.161 (GigabitEthernet1), destination 25.7.73.166
```

**show crypto ikev2 sa:**
```
Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         25.7.73.161/500       25.7.73.166/500       none/none            READY
      Encr: AES-CBC, keysize: 256, PRF: SHA256, Hash: SHA256, DH Grp:14, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/60 sec
```

**show crypto ipsec sa (fragmento):**
```
local  ident (addr/mask/prot/port): (25.7.73.161/255.255.255.255/47/0)
remote ident (addr/mask/prot/port): (25.7.73.166/255.255.255.255/47/0)
#pkts encaps: 5, #pkts encrypt: 5, #pkts digest: 5
#pkts decaps: 4, #pkts decrypt: 4, #pkts verify: 4
in use settings ={Transport, }
Status: ACTIVE(ACTIVE)
```

> Los identificadores muestran protocolo `47` (GRE) — confirma que IPSec está cifrando los paquetes GRE entre las IPs WAN, no el tráfico IP de las LANs directamente.

### Pruebas de conectividad

**Ping PC1 → PC2:**
```
PC1> ping 25.7.73.178
25.7.73.178 icmp_seq=1 timeout
84 bytes from 25.7.73.178 icmp_seq=2 ttl=62 time=474.596 ms
84 bytes from 25.7.73.178 icmp_seq=3 ttl=62 time=55.298 ms
84 bytes from 25.7.73.178 icmp_seq=4 ttl=62 time=105.548 ms
84 bytes from 25.7.73.178 icmp_seq=5 ttl=62 time=47.365 ms
```

> El primer paquete dio timeout durante la negociación IKEv2 (IKE_SA_INIT + IKE_AUTH) y el establecimiento del túnel GRE. Los paquetes subsiguientes viajan cifrados exitosamente. Este comportamiento es idéntico al observado en P3 con IKEv1.

---

## Capturas de pantalla

> Insertar aquí las capturas de:
> 1. Topología completa en GNS3 con nombre y matrícula visible
![](./Topologia.png)

> 2. `show interface Tunnel0` en R1 y R2 (GRE/IP, up/up)
![](./C1R1.png)
![](./C1R2.png)

> 3. `show crypto ikev2 sa` en R1 y R2 (Status READY)
![](./C2R1.png)
![](./C2R2.png)

> 4. `show crypto ipsec sa` en R1 y R2 (modo Transport, protocolo 47, Status ACTIVE)
![](./C3R1.png)
![](./C3R2.png)

> 5. `show ip route` en R1 y R2 (ruta via Tunnel0)
![](./C4R1.png)
![](./C4R2.png)

> 6. Ping PC1 → PC2 (primer timeout y luego exitoso)
![](./PING1.png)

> 7. Ping PC2 → PC1
![](./PING2.png)

---

## Conclusión

Se configuró exitosamente una VPN IPSec IKEv2 Site-to-Site con túnel GRE (GRE over IPSec) entre R1 y R2. La combinación GRE + IPSec IKEv2 en modo Transport representa la implementación más moderna de esta arquitectura: GRE encapsula el tráfico entre las LANs (incluyendo soporte nativo de multicast/broadcast y protocolos de enrutamiento dinámico), mientras que IKEv2 negocia las Security Associations de forma más eficiente que IKEv1 (4 mensajes vs 6-9), y el modo Transport de IPSec cifra el payload GRE sin re-encapsularlo. La SA IKEv2 alcanzó el estado READY con AES-CBC-256, SHA-256 y DH Group 14, y la conectividad entre LAN-A (25.7.73.168/29) y LAN-B (25.7.73.176/29) fue verificada exitosamente mediante ping bidireccional.
