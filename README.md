# 🚀 GUÍA COMPLETA DE DESPLIEGUE: CLUSTER RoCEv2 Y TELEMETRÍA

> **Nota importante:** Sigue este orden de terminales en tu entorno WSL. Sitúate siempre en el directorio base `~/qemu-roce` antes de ejecutar los comandos.

---

## FASE 1: Preparación del Host (WSL)

### 1.1 Crear el Script de Red

Abre la **Terminal 1**, ejecuta `nano start-topology.sh` y pega el siguiente código:

```bash
#!/bin/bash
set -e

echo "=== Creando bridge y TAPs ==="
sudo ip link add br-roce type bridge 2>/dev/null || true
sudo ip link set br-roce up

for i in 0 1 2 10; do
    sudo ip tuntap add dev tap$i mode tap 2>/dev/null || true
    sudo ip link set tap$i master br-roce
    sudo ip link set tap$i up
done

sudo ip addr add 10.10.0.254/24 dev br-roce 2>/dev/null || true

echo "=== Configurando iptables ==="
sudo sysctl -w net.bridge.bridge-nf-call-iptables=0 > /dev/null
sudo sysctl -w net.bridge.bridge-nf-call-ip6tables=0 > /dev/null
sudo iptables -I FORWARD -i br-roce -o br-roce -j ACCEPT 2>/dev/null || true
sudo sysctl -w net.ipv4.ip_forward=1 > /dev/null

echo "=== Topología lista ==="
bridge link show
