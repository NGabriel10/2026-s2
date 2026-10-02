# Laboratorio 1 — Redes virtuales con Linux

## 1. Datos del entorno

- **Usuario:** norman
- **Distribución:** Linux LAPTOP-7KG719CJ 6.18.40.1-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Fri Jul 31 22:12:15 UTC 2026 x86_64 GNU/Linux
- **Kernel:** PRETTY_NAME="Ubuntu 26.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="26.04"
VERSION="26.04.1 LTS (Resolute Raccoon)"
VERSION_CODENAME=resolute
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=resolute
LOGO=ubuntu-logo

## 2. Objetivo

Crear dos namespaces (hostA y hostB) conectados mediante un par veth y verificar conectividad IPv4.

## 3. Topología 
Se construyó una red virtual aislada con dos namespaces:

``
        Linux (WSL2)
           │
    ┌──────┴──────┐
    ▼             ▼
  hostA         hostB
namespace     namespace
    │             │
  vethA ═════ vethB
    │             │
10.10.1.1/30  10.10.1.2/30

## 4. Construcción

Se crearon los namespace:
sudo ip netns add hostA
sudo ip netns add hostB
ip netns list

Se creo el par veth:
sudo ip link add vethA type veth peer name vethB
ip link

Se movieron las interfaces a sus respectivos namespace
sudo ip link set vethA netns hostA
sudo ip link set vethB netns hostB
sudo ip netns exec hostA ip link
sudo ip netns exec hostB ip link

Se configuraron las direcciones IP:
sudo ip netns exec hostA ip addr add 10.10.1.1/30 dev vethA
sudo ip netns exec hostB ip addr add 10.10.1.2/30 dev vethB
sudo ip netns exec hostA ip addr
sudo ip netns exec hostB ip addr

Se levantaron las interfaces
sudo ip netns exec hostA ip link set lo up
sudo ip netns exec hostA ip link set vethA up
sudo ip netns exec hostB ip link set lo up
sudo ip netns exec hostB ip link set vethB up

Se verificaron las rutas automáticas generadas por Linux:
sudo ip netns exec hostA ip route
sudo ip netns exec hostB ip route

La Salida mostro:
10.10.1.0/30 dev vethA proto kernel scope link src 10.10.1.1
10.10.1.0/30 dev vethB proto kernel scope link src 10.10.1.2
## 5. Verificación

Se comprobo la conectividad con ping en ambos hosts:
sudo ip netns exec hostA ping -c 4 10.10.1.2
sudo ip netns exec hostB ping -c 4 10.10.1.1

El Resultado esperado fue:
4 packets transmitted, 4 received, 0% packet loss

Se reviso vecinos ARP:
sudo ip netns exec hostA ip neigh
sudo ip netns exec hostB ip neigh

Obtuvimos la salida:
10.10.1.2 dev vethA lladdr ba:8d:ee:28:57:1f STALE
10.10.1.1 dev vethB lladdr 3a:a4:19:e4:1a:72 STALE
con esto confirmamos que el host conoce la direccion MAC del otro extremo

Se hizo una captura de trafico con : 
sudo ip netns exec hostA tcpdump -n -i vethA

y en el host B se puso:
sudo ip netns exec hostB ping -c 4 10.10.1.1
se observaron trafico icmp y arp

## 6. Falla y diagnóstico
Se ejecuto una falla controlada apagando veth B en host B :
sudo ip netns exec hostB ip link set vethB down
sudo ip netns exec hostA ping -c 4 10.10.1.2
se observo:
From 10.10.1.1 icmp_seq=1 Destination Host Unreachable
4 packets transmitted, 0 received, 100% packet loss
Se diagnostico con:
sudo ip netns exec hostA ip link
sudo ip netns exec hostA ip addr
sudo ip netns exec hostA ip route
sudo ip netns exec hostA ip neigh

sudo ip netns exec hostB ip link
sudo ip netns exec hostB ip addr
sudo ip netns exec hostB ip route
Observaciones:

En hostB, la interfaz vethB aparece como state DOWN.
En hostA, la interfaz vethA muestra NO-CARRIER y state DOWN.
La ruta sigue existiendo, pero el enlace está caído.
El error fue por estado de la interfaz, no por una dirección IP incorrecta
Se la restauro con:
sudo ip netns exec hostB ip link set vethB up
sudo ip netns exec hostA ping -c 4 10.10.1.2
Entonces el resultado fue:
4 packets transmitted, 4 received, 0% packet loss
Demostrando que el error fue el enlace apagado

## 7. Conclusiones
los namespace tiene su propria mascara de red
el par veth representa los enlaces
la configuracion ipv4 permitio la comunicacion
el uso de ping verifico esto
el protocolo arp tambien perimito esto
la falla controlada demostro el estado down que provoca perdida total de comunicacion
Este laboratio permitio simular un entorno virutal de redes aisladas y conectadas por un enlace virtual
