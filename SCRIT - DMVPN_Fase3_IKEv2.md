# DMVPN Fase 3 — IKEv2

**Topología:** R1 = Hub · R2 = Spoke1 · R3 = Spoke2  
**Enrutamiento dinámico:** EIGRP AS 100  
**Túnel mGRE:** `172.16.0.0/24` — Hub=`.1` · R2=`.2` · R3=`.3`

---

## Direccionamiento

| Dispositivo | Interfaz | IP              | Rol                  |
|-------------|----------|-----------------|----------------------|
| R1 Hub      | e0/0     | 200.1.15.1/24   | NBMA fuente Tunnel10 |
| R1 Hub      | e0/1     | 200.1.99.1/24   | Tránsito a R3        |
| R2 Spoke1   | e0/0     | 200.1.15.2/24   | NBMA                 |
| R2 Spoke1   | e0/1     | 10.15.99.1/24   | LAN                  |
| R3 Spoke2   | e0/0     | 200.1.99.2/24   | NBMA                 |
| R3 Spoke2   | e0/1     | 192.168.99.1/24 | LAN                  |
| Linux6      | e0       | 10.15.99.2/24   | Host LAN R2          |
| VPC8        | eth0     | 10.15.99.3/24   | Host LAN R2          |
| Linux7      | e0       | 192.168.99.2/24 | Host LAN R3          |
| VPC9        | eth0     | 192.168.99.3/24 | Host LAN R3          |

---

## Diferencias clave vs Fase 2

| Aspecto              | Fase 2 — IKEv1                  | Fase 3 — IKEv2                        |
|----------------------|----------------------------------|---------------------------------------|
| Negociación IKE      | `crypto isakmp policy`           | `crypto ikev2 proposal` + `policy`    |
| Clave PSK            | `crypto isakmp key`              | `crypto ikev2 keyring`                |
| Perfil de identidad  | No existe                        | `crypto ikev2 profile` (obligatorio)  |
| Perfil IPSec         | solo `set transform-set`         | + `set ikev2-profile`                 |
| Hub Tunnel10         | sin `nhrp redirect`              | `ip nhrp redirect`                    |
| Spoke Tunnel10       | sin `nhrp shortcut`              | `ip nhrp shortcut`                    |
| Hub puede resumir    | No                               | Sí (NHRP gestiona el atajo)           |
| Verificar IKE        | `show crypto isakmp sa`          | `show crypto ikev2 sa`                |

---

## Paso 1 — Interfaces físicas y rutas

### R1 Hub
```
conf t
interface e0/0
 ip address 200.1.15.1 255.255.255.0
 no shut
interface e0/1
 ip address 200.1.99.1 255.255.255.0
 no shut
```

### R2 Spoke1
```
conf t
interface e0/0
 ip address 200.1.15.2 255.255.255.0
 no shut
interface e0/1
 ip address 10.15.99.1 255.255.255.0
 no shut
ip route 0.0.0.0 0.0.0.0 200.1.15.1
```

### R3 Spoke2
```
conf t
interface e0/0
 ip address 200.1.99.2 255.255.255.0
 no shut
interface e0/1
 ip address 192.168.99.1 255.255.255.0
 no shut
ip route 0.0.0.0 0.0.0.0 200.1.99.1
```

---

## Paso 2 — Propuesta IKEv2

> Reemplaza `crypto isakmp policy`. Se definen cifrado, integridad y grupo DH.

### R1, R2 y R3 (igual en todos)
```
crypto ikev2 proposal DMVPN-PROP
 encryption aes-cbc-128
 integrity sha1
 group 2
```

---

## Paso 3 — Política IKEv2

> Referencia la propuesta. Equivale al número de prioridad del `isakmp policy` de IKEv1.

### R1, R2 y R3 (igual en todos)
```
crypto ikev2 policy DMVPN-POL
 proposal DMVPN-PROP
```

---

## Paso 4 — Keyring IKEv2

> Reemplaza `crypto isakmp key`. El Hub acepta cualquier Spoke con `0.0.0.0 0.0.0.0`.  
> Los Spokes apuntan a la IP fuente del Tunnel10 del Hub (`200.1.15.1`).

### R1 Hub
```
crypto ikev2 keyring DMVPN-KEYS
 peer SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key cisco123
```

### R2 Spoke1
```
crypto ikev2 keyring DMVPN-KEYS
 peer HUB
  address 200.1.15.1
  pre-shared-key cisco123
```

### R3 Spoke2
```
crypto ikev2 keyring DMVPN-KEYS
 peer HUB
  address 200.1.15.1
  pre-shared-key cisco123
```

---

## Paso 5 — Perfil IKEv2

> No existe equivalente en IKEv1. Une keyring + política y define la identidad local/remota.

### R1 Hub
```
crypto ikev2 profile DMVPN-IKE
 match identity remote address 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN-KEYS
```

### R2 Spoke1
```
crypto ikev2 profile DMVPN-IKE
 match identity remote address 200.1.15.1
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN-KEYS
```

### R3 Spoke2
```
crypto ikev2 profile DMVPN-IKE
 match identity remote address 200.1.15.1
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN-KEYS
```

---

## Paso 6 — IPSec transform-set y perfil IPSec

> El perfil IPSec en IKEv2 **debe referenciar** el perfil IKEv2 con `set ikev2-profile`.  
> Sin esa línea IKEv2 no se usa aunque esté configurado.

### R1, R2 y R3 (igual en todos)
```
crypto ipsec transform-set DMVPN-SET esp-aes esp-sha-hmac
 mode transport

crypto ipsec profile DMVPN-PROF
 set transform-set DMVPN-SET
 set ikev2-profile DMVPN-IKE
```

---

## Paso 7 — Interfaz Tunnel10 mGRE + NHRP Fase 3

> **Diferencia crítica vs Fase 2:**  
> - Hub agrega `ip nhrp redirect` — informa al Spoke que existe un atajo directo.  
> - Spokes agregan `ip nhrp shortcut` — instalan la ruta de atajo en su tabla de routing.

### R1 Hub
```
interface Tunnel10
 ip address 172.16.0.1 255.255.255.0
 no ip redirects
 ip nhrp network-id 100
 ip nhrp map multicast dynamic
 ip nhrp authentication dmvpn1
 ip nhrp redirect
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
```

### R2 Spoke1
```
interface Tunnel10
 ip address 172.16.0.2 255.255.255.0
 no ip redirects
 ip nhrp network-id 100
 ip nhrp authentication dmvpn1
 ip nhrp nhs 172.16.0.1
 ip nhrp map 172.16.0.1 200.1.15.1
 ip nhrp map multicast 200.1.15.1
 ip nhrp registration no-unique
 ip nhrp shortcut
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
```

### R3 Spoke2
```
interface Tunnel10
 ip address 172.16.0.3 255.255.255.0
 no ip redirects
 ip nhrp network-id 100
 ip nhrp authentication dmvpn1
 ip nhrp nhs 172.16.0.1
 ip nhrp map 172.16.0.1 200.1.15.1
 ip nhrp map multicast 200.1.15.1
 ip nhrp registration no-unique
 ip nhrp shortcut
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
```

---

## Paso 8 — EIGRP dinámico

> En Fase 3 el Hub **sí puede resumir** rutas sobre Tunnel10 porque NHRP gestiona el atajo.  
> El `no ip split-horizon eigrp 100` sigue siendo obligatorio en el Hub.

### R1 Hub
```
router eigrp 100
 network 172.16.0.0 0.0.0.255
 network 10.15.99.0 0.0.0.255
 network 192.168.99.0 0.0.0.255
 no auto-summary

interface Tunnel10
 no ip split-horizon eigrp 100
```

### R2 Spoke1
```
router eigrp 100
 network 172.16.0.0 0.0.0.255
 network 10.15.99.0 0.0.0.255
 no auto-summary
```

### R3 Spoke2
```
router eigrp 100
 network 172.16.0.0 0.0.0.255
 network 192.168.99.0 0.0.0.255
 no auto-summary
```

---

## Paso 9 — Hosts

### Linux6 (e0)
```
ip addr add 10.15.99.2/24 dev eth0
ip route add default via 10.15.99.1
```

### Linux7 (e0)
```
ip addr add 192.168.99.2/24 dev eth0
ip route add default via 192.168.99.1
```

### VPC8 (eth0)
```
ip 10.15.99.3 255.255.255.0 10.15.99.1
```

### VPC9 (eth0)
```
ip 192.168.99.3 255.255.255.0 192.168.99.1
```

---

## Paso 10 — Verificación

### Estado IKEv2
```
show crypto ikev2 sa
show crypto ikev2 session detail
```
Debe mostrar `READY` para cada Spoke.

### Estado NHRP y túnel
```
show dmvpn detail
show ip nhrp
show ip nhrp shortcut
```

**Resultado esperado en R1:**
```
172.16.0.2/32 via 172.16.0.2 — Type: dynamic, Flags: registered nhop — NBMA: 200.1.15.2
172.16.0.3/32 via 172.16.0.3 — Type: dynamic, Flags: registered nhop — NBMA: 200.1.99.2
```

`show ip nhrp shortcut` en los Spokes muestra las rutas de atajo Spoke↔Spoke instaladas.

### IPSec
```
show crypto ipsec sa
```
> Los contadores `encaps/decaps` de la SA Spoke↔Spoke comienzan en 0 y se activan
> solo cuando hay tráfico entre ellos — es comportamiento normal en Fase 3.

### EIGRP
```
show ip eigrp neighbors
show ip route eigrp
```
R1 debe ver a R2 y R3 como vecinos. R2 debe aprender `192.168.99.0/24` y R3 debe aprender `10.15.99.0/24`.

### Conectividad
```
ping 192.168.99.1 source 10.15.99.1
ping 192.168.99.2 source 10.15.99.1
ping 192.168.99.3 source 10.15.99.1
ping 10.15.99.2 source 192.168.99.1
```

---

## Solución de problemas — errores encontrados en laboratorio

### R2 no se registra en NHRP (Type: incomplete, Flags: negative)

```
# En R2 — reescribir bloque NHRP en Tunnel10
interface Tunnel10
 no ip nhrp authentication dmvpn1
 ip nhrp authentication dmvpn1
 no ip nhrp nhs 172.16.0.1
 no ip nhrp map 172.16.0.1 200.1.15.1
 no ip nhrp map multicast 200.1.15.1
 ip nhrp nhs 172.16.0.1
 ip nhrp map 172.16.0.1 200.1.15.1
 ip nhrp map multicast 200.1.15.1
 ip nhrp registration no-unique
 ip nhrp shortcut

# En R1 — limpiar entrada negativa
clear ip nhrp 172.16.0.2
```

### R3 no anuncia su LAN por EIGRP

```
# En R3
router eigrp 100
 network 192.168.99.0 0.0.0.255
 network 172.16.0.0 0.0.0.255
```

### SA Spoke↔Spoke con pkts encaps: 0

Comportamiento normal. Los túneles directos en Fase 3 se crean bajo demanda.
Forzar activación con:
```
ping 192.168.99.1 source 10.15.99.1 repeat 100
show ip nhrp shortcut
show crypto ipsec sa peer 200.1.99.2
```

---

## Notas importantes — Fase 3

- `ip nhrp redirect` en el Hub y `ip nhrp shortcut` en los Spokes son las **dos líneas que definen Fase 3**. Sin ellas, funciona como Fase 1.
- Todos los routers deben tener exactamente la misma `ip nhrp authentication` y el mismo `ip nhrp network-id`.
- El perfil IPSec **debe incluir** `set ikev2-profile DMVPN-IKE` — sin esa línea IKEv2 no participa en la protección del túnel.
- R3 usa como gateway `200.1.99.1` (e0/1 de R1) pero el Hub tiene `tunnel source e0/0` (`200.1.15.1`). R3 registra su IP NBMA con el Hub atravesando R1 internamente.
