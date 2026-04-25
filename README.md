🚀 GUÍA COMPLETA DE DESPLIEGUE: CLUSTER RoCEv2 Y TELEMETRÍA
Sigue este orden de terminales en tu WSL. [cite_start]Sitúate siempre en el directorio de trabajo ~/qemu-roce[cite: 2].

## Índice
[cite_start]FASE 1: Preparación del Host (WSL) [cite: 3]
[cite_start]FASE 2: Arranque de la Topología [cite: 66]
[cite_start]FASE 3: Activación de Red (Consola QEMU) [cite: 73]
[cite_start]FASE 4: Gestión y Telemetría (SSH) [cite: 134]
[cite_start]FASE 5: Verificación [cite: 182]
## FASE 1: Preparación del Host (WSL)
### 1.1 Crear el Script de Red
[cite_start]En la Terminal 1, ejecuta nano start-topology.sh y pega este código[cite: 4, 5]:

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
[cite: 7-24, 193-210]

[cite_start]Cierra y guarda (Ctrl+O, Enter, Ctrl+X)[cite: 25]. [cite_start]Dale permisos: chmod +x start-topology.sh[cite: 25].

### 1.2 Crear el Script para VM3
[cite_start]En la Terminal 1, ejecuta nano start-mv3.sh y pega este código[cite: 26, 27]:

#!/bin/bash
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
manage_etc_hosts: true
users:
  - name: user
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: false
    plain_text_passwd: "rdma"
ssh_pwauth: true
EOF

cat <<EOF > "$BASE_DIR/cloud-init/$VM_NAME/meta-data"
instance-id: $VM_NAME
local-hostname: $VM_NAME
EOF

genisoimage -quiet -output "$BASE_DIR/seed-$VM_NAME.iso" -volid cidata -joliet -rock \
    "$BASE_DIR/cloud-init/$VM_NAME/user-data" \
    "$BASE_DIR/cloud-init/$VM_NAME/meta-data"

if [ ! -f "$BASE_DIR/$VM_NAME.qcow2" ]; then
    qemu-img create -f qcow2 -b "$BASE_DIR/base-roce.qcow2" -F qcow2 "$BASE_DIR/$VM_NAME.qcow2"
fi

sudo qemu-system-x86_64 -name $VM_NAME -machine q35,accel=kvm -cpu host -m 2048 -smp 2 \
    -drive file="$BASE_DIR/$VM_NAME.qcow2",format=qcow2,if=virtio \
    -drive file="$BASE_DIR/seed-$VM_NAME.iso",format=raw,if=virtio \
    -netdev tap,id=net0,ifname=$TAP_IF,script=no,downscript=no \
    -device virtio-net-pci,netdev=net0,mac=$MAC_ADDR -nographic -serial mon:stdio
[cite: 29-64, 215-250]

[cite_start]Cierra y guarda[cite: 65]. [cite_start]Dale permisos: chmod +x start-mv3.sh[cite: 65].

## FASE 2: Arranque de la Topología
[cite_start]Abre 5 pestañas de terminal y ejecuta un comando en cada una[cite: 66, 67]:

[cite_start]Terminal 1: sudo ./start-topology.sh [cite: 68]
[cite_start]Terminal 2 (VM1): ./start-vm.sh 1 [cite: 69]
[cite_start]Terminal 3 (VM2): ./start-vm.sh 2 [cite: 70]
[cite_start]Terminal 4 (VM3): ./start-mv3.sh [cite: 71]
[cite_start]Terminal 5 (Switch): ./start-switch.sh [cite: 72]
## FASE 3: Activación de Red (Consola QEMU)
[cite_start]Inicia sesión en las ventanas de QEMU con usuario: user / pass: rdma[cite: 73, 74].

### 3.1 Configuración de Workers (VM1, VM2, VM3)
[cite_start]En cada máquina, ejecuta nano setup-roce.sh y pega esto[cite: 75, 76]:

#!/bin/bash
sudo apt update && sudo apt install snmpd snmp -y

sudo bash -c 'cat > /etc/snmp/snmpd.conf' << 'EOF'
agentAddress udp:161
rocommunity public 10.10.0.254
rocommunity public localhost
pass_persist .1.3.6.1.4.1.99999 /usr/local/bin/roce_agent.py
EOF

sudo bash -c 'cat > /usr/local/bin/roce_agent.py' << 'EOF'
#!/usr/bin/env python3
import sys, os
BASE_OID, IFACE = ".1.3.6.1.4.1.99999", "enp0s2"
def read_val(f):
    try:
        with open(f"/sys/class/net/{IFACE}/statistics/{f}", "r") as f: return f.read().strip()
    except: return "0"
def get_metrics():
    m = {f"{BASE_OID}.1.{i}": "0" for i in range(1,10)}
    m[f"{BASE_OID}.1.1"], m[f"{BASE_OID}.1.2"] = read_val("tx_bytes"), read_val("rx_bytes")
    m[f"{BASE_OID}.1.3"], m[f"{BASE_OID}.1.4"] = read_val("tx_packets"), read_val("rx_packets")
    try:
        with open("/proc/net/netstat", "r") as f:
            lines = f.readlines()
            for i, line in enumerate(lines):
                if line.startswith("IpExt:"):
                    k = line.split(); v = lines[i+1].split()
                    m[f"{BASE_OID}.2.1"] = v[k.index("InCEPkts")]
                    m[f"{BASE_OID}.2.4"] = v[k.index("InNoECTPkts")]
    except: pass
    return m
def main():
    sys.stdout = os.fdopen(sys.stdout.fileno(), "w", 1)
    while True:
        l = sys.stdin.readline()
        if not l or "PING" in l:
            if l: print("PONG")
            continue
        oid = sys.stdin.readline().strip()
        metrics = get_metrics()
        if "getnext" in l:
            for k in sorted(metrics.keys()):
                if k > oid:
                    print(k); print("counter64"); print(metrics[k]); break
            else: print("NONE")
        else:
            if oid in metrics: print(oid); print("counter64"); print(metrics[oid])
            else: print("NONE")
if __name__ == "__main__": main()
EOF
sudo chmod +x /usr/local/bin/roce_agent.py
sudo systemctl restart snmpd
[cite: 78-128, 264-314]

[cite_start]Ejecuta: chmod +x setup-roce.sh && ./setup-roce.sh[cite: 129].

### 3.2 Activación del Switch (Terminal 5)
[cite_start]En la consola del switch, habilita la IP de gestión[cite: 130, 131]:

sudo ip addr add 10.10.0.10/24 dev enp0s2 && sudo ip link set enp0s2 up
[cite_start][cite: 133]

## FASE 4: Gestión y Telemetría (SSH)
[cite_start]Abre 4 terminales nuevas en WSL y conéctate[cite: 134, 135]:

[cite_start]T6: ssh user@10.10.0.1 [cite: 136]
[cite_start]T7: ssh user@10.10.0.2 [cite: 137]
[cite_start]T8: ssh user@10.10.0.3 [cite: 138]
[cite_start]T9: ssh user@10.10.0.10 [cite: 139]
### 4.1 Configuración Final del Switch (Terminal 9)
[cite_start]Dentro del SSH del Switch, configura el agente OVS[cite: 140, 141]:

[cite_start]sudo apt update && sudo apt install openvswitch-switch snmpd -y [cite: 142]
[cite_start]sudo ovs-vsctl add-br br-roce [cite: 143]
[cite_start]sudo nano /etc/snmp/snmpd.conf -> Borra todo y pega[cite: 144]:
agentAddress udp:161
rocommunity public 10.10.0.254
rocommunity public localhost
pass_persist .1.3.6.1.4.1.99999.3 /usr/local/bin/ovs_agent.py
[cite: 146-149, 332-335] 4. [cite_start]sudo nano /usr/local/bin/ovs_agent.py -> Pega esto[cite: 150]:

#!/usr/bin/env python3
import sys, os, subprocess, re
BASE_OID = ".1.3.6.1.4.1.99999.3"
def get_metrics():
    m = {f"{BASE_OID}.{i}": "0" for i in range(1, 8)}
    try:
        res = subprocess.check_output(["sudo", "ovs-ofctl", "dump-ports", "br-roce"]).decode()
        m[f"{BASE_OID}.1"] = str(sum(map(int, re.findall(r"rx.*?bytes[:=](\d+)", res, re.I|re.S))))
        m[f"{BASE_OID}.2"] = str(sum(map(int, re.findall(r"tx.*?bytes[:=](\d+)", res, re.I|re.S))))
        m[f"{BASE_OID}.3"] = str(sum(map(int, re.findall(r"rx.*?pkts[:=](\d+)", res, re.I|re.S))))
        m[f"{BASE_OID}.4"] = str(sum(map(int, re.findall(r"tx.*?pkts[:=](\d+)", res, re.I|re.S))))
    except: pass
    return m
def main():
    sys.stdout = os.fdopen(sys.stdout.fileno(), 'w', 1)
    while True:
        line = sys.stdin.readline()
        if not line: break
        if "PING" in line: print("PONG"); continue
        oid_req = sys.stdin.readline().strip()
        metrics = get_metrics()
        if "getnext" in line:
            for k in sorted(metrics.keys()):
                if k > oid_req: print(k); print("counter64"); print(metrics[k]); break
            else: print("NONE")
        else:
            if oid_req in metrics: print(oid_req); print("counter64"); print(metrics[oid_req])
            else: print("NONE")
if __name__ == "__main__": main()
