🚀 GUÍA DE DESPLIEGUE: CLUSTER RoCEv2 Y TELEMETRÍA EN WSL2Guía técnica detallada para la emulación de redes RDMA utilizando Soft-RoCE sobre QEMU/KVM en entornos Windows Subsystem for Linux (WSL2).## ÍndiceContexto y JustificaciónRequisitos Previos y HardwareFASE 1: Preparación del Host (WSL)FASE 2: Arranque de la TopologíaFASE 3: Activación de Red (Consola QEMU)FASE 4: Gestión y Telemetría (SSH)FASE 5: Verificación de Tráfico## 1. Contexto y Justificación### ¿Por qué máquinas virtuales (QEMU)?El driver rdma_rxe requiere instancias de kernel independientes para simular nodos RDMA reales. A diferencia de los contenedores o namespaces de red, cada VM posee su propia pila de red y socket UDP:4791, permitiendo la generación de tráfico RoCEv2 real capturable en el bridge del host.## 2. Requisitos Previos y Hardware### Hardware MínimoCPU: Soporte Intel VT-x o AMD-V habilitado en BIOS.RAM: Mínimo 16 GB recomendado.Disco: 15 GB de espacio libre.### Configuración de WSL2Es necesario habilitar la virtualización anidada. Edita el archivo C:\Users\<USUARIO>\.wslconfig:[wsl2]
nestedVirtualization=true
memory=8GB
processors=4
Reinicia WSL desde PowerShell:wsl --shutdown
Verifica el acceso a KVM desde la terminal de WSL:ls -la /dev/kvm
## FASE 1: Preparación del Host (WSL)### 1.1 Script de Red (Bridge + TAPs)Crea el script para levantar el switch virtual: nano start-topology.sh.#!/bin/bash
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
### 1.2 Script de Arranque para VM3Crea el script de inicio: nano start-mv3.sh.#!/bin/bash
BASE_DIR="$HOME/qemu-roce"
VM_NAME="vm3"
TAP_IF="tap2"
MAC_ADDR="52:54:00:00:00:03"

sudo ip tuntap add dev $TAP_IF mode tap 2>/dev/null || true
sudo ip link set $TAP_IF master br-roce
sudo ip link set $TAP_IF up

mkdir -p "$BASE_DIR/cloud-init/$VM_NAME"
cat <<EOF > "$BASE_DIR/cloud-init/$VM_NAME/user-data"
#cloud-config
hostname: $VM_NAME
users:
  - name: user
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: false
    plain_text_passwd: "rdma"
ssh_pwauth: true
EOF

genisoimage -quiet -output "$BASE_DIR/seed-$VM_NAME.iso" -volid cidata -joliet -rock \
    "$BASE_DIR/cloud-init/$VM_NAME/user-data"

sudo qemu-system-x86_64 -name $VM_NAME -machine q35,accel=kvm -cpu host -m 2048 -smp 2 \
    -drive file="$BASE_DIR/$VM_NAME.qcow2",format=qcow2,if=virtio \
    -drive file="$BASE_DIR/seed-$VM_NAME.iso",format=raw,if=virtio \
    -netdev tap,id=net0,ifname=$TAP_IF,script=no,downscript=no \
    -device virtio-net-pci,netdev=net0,mac=$MAC_ADDR -nographic -serial mon:stdio
## FASE 2: Arranque de la TopologíaAbre terminales independientes para cada proceso:Host: sudo ./start-topology.shVM1: ./start-vm.sh 1VM2: ./start-vm.sh 2VM3: ./start-mv3.shSwitch: ./start-switch.sh## FASE 3: Activación de Red (Consola QEMU)Logueate en las VMs (user: user / pass: rdma) y configura la telemetría local:### 3.1 Script setup-roce.shEjecuta esto en los Workers:#!/bin/bash
sudo apt update && sudo apt install snmpd snmp -y

sudo bash -c 'cat > /etc/snmp/snmpd.conf' << 'EOF'
agentAddress udp:161
rocommunity public 10.10.0.254
pass_persist .1.3.6.1.4.1.99999 /usr/local/bin/roce_agent.py
EOF

# Reiniciar servicio
sudo systemctl restart snmpd
## FASE 4: Gestión y Telemetría (SSH)Conéctate desde tu WSL principal:T6 (VM1): ssh user@10.10.0.1T9 (Switch): ssh user@10.10.0.10### 4.1 Configuración OVS en el SwitchInstala y configura el agente para los puertos del switch:#!/usr/bin/env python3
import sys, os, subprocess, re
BASE_OID = ".1.3.6.1.4.1.99999.3"

def get_metrics():
    m = {f"{BASE_OID}.{i}": "0" for i in range(1, 8)}
    try:
        res = subprocess.check_output(["sudo", "ovs-ofctl", "dump-ports", "br-roce"]).decode()
        m[f"{BASE_OID}.1"] = str(sum(map(int, re.findall(r"rx.*?bytes[:=](\d+)", res, re.I|re.S))))
        m[f"{BASE_OID}.2"] = str(sum(map(int, re.findall(r"tx.*?bytes[:=](\d+)", res, re.I|re.S))))
    except: pass
    return m

if __name__ == "__main__":
    # Bucle principal para SNMP pass_persist
    # ... (lógica de lectura de STDIN)
## FASE 5: Verificación de TráficoGenerar Tráfico:# En VM2
ibv_rc_pingpong -d rxe0 -g 1
# En VM1
ibv_rc_pingpong -d rxe0 -g 1 10.10.0.2
Capturar Métricas:snmpwalk -v2c -c public 10.10.0.1 .1.3.6.1.4.1.99999
