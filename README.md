# azure-openvpn

#!/bin/bash
#
# Azure-friendly OpenVPN installer
# Based on Nyr/openvpn-install behavior, but ALWAYS asks for public IP/hostname.
#
# Usage:
#   sudo bash azure-openvpn-manual-ip-install.sh
#

set -e

if [[ "$EUID" -ne 0 ]]; then
  echo "Please run as root:"
  echo "sudo bash $0"
  exit 1
fi

if ! command -v curl >/dev/null 2>&1; then
  apt-get update
  apt-get install -y curl
fi

if ! command -v wget >/dev/null 2>&1; then
  apt-get update
  apt-get install -y wget
fi

echo "=============================================="
echo " Azure OpenVPN Installer - Manual Public IP"
echo "=============================================="
echo

PRIVATE_IP=$(ip -4 addr show scope global | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -n 1)

if [[ -z "$PRIVATE_IP" ]]; then
  echo "Could not detect private IP."
  exit 1
fi

DETECTED_PUBLIC_IP=$(curl -4 -s ifconfig.me || true)

echo "Detected private IP inside VM: $PRIVATE_IP"
echo "Detected public IP from internet: ${DETECTED_PUBLIC_IP:-not detected}"
echo
echo "IMPORTANT:"
echo "For Azure, OpenVPN server should bind to private IP:"
echo "  $PRIVATE_IP"
echo
echo "But client .ovpn must use your Azure Public IP."
echo

PUBLIC_IP=""
while [[ -z "$PUBLIC_IP" ]]; do
  read -rp "Enter Azure Public IP or DNS hostname manually: " PUBLIC_IP
done

read -rp "OpenVPN port [1194]: " PORT
PORT=${PORT:-1194}

read -rp "Protocol udp/tcp [udp]: " PROTOCOL
PROTOCOL=${PROTOCOL:-udp}

read -rp "Client name [client]: " CLIENT
CLIENT=${CLIENT:-client}

echo
echo "Installing OpenVPN..."
echo "Private bind IP : $PRIVATE_IP"
echo "Public client IP: $PUBLIC_IP"
echo "Port            : $PORT"
echo "Protocol        : $PROTOCOL"
echo "Client name     : $CLIENT"
echo

apt-get update
apt-get install -y openvpn easy-rsa iptables openssl ca-certificates

mkdir -p /etc/openvpn/server/easy-rsa
cp -r /usr/share/easy-rsa/* /etc/openvpn/server/easy-rsa/

cd /etc/openvpn/server/easy-rsa

./easyrsa --batch init-pki
./easyrsa --batch build-ca nopass
./easyrsa --batch gen-req server nopass
./easyrsa --batch sign-req server server
./easyrsa --batch gen-req "$CLIENT" nopass
./easyrsa --batch sign-req client "$CLIENT"
./easyrsa gen-dh
openvpn --genkey secret ta.key

cp pki/ca.crt /etc/openvpn/server/
cp pki/issued/server.crt /etc/openvpn/server/
cp pki/private/server.key /etc/openvpn/server/
cp pki/dh.pem /etc/openvpn/server/
cp ta.key /etc/openvpn/server/

cat > /etc/openvpn/server/server.conf <<EOF
local $PRIVATE_IP
port $PORT
proto $PROTOCOL
dev tun

ca ca.crt
cert server.crt
key server.key
dh dh.pem

server 10.8.0.0 255.255.255.0
topology subnet

push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 8.8.8.8"

keepalive 10 120
cipher AES-256-GCM
auth SHA256
tls-auth ta.key 0

user nobody
group nogroup
persist-key
persist-tun
verb 3
EOF

echo 1 > /proc/sys/net/ipv4/ip_forward
cat > /etc/sysctl.d/99-openvpn-forward.conf <<EOF
net.ipv4.ip_forward=1
EOF

iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE || true
iptables -I INPUT -p "$PROTOCOL" --dport "$PORT" -j ACCEPT || true
iptables -I FORWARD -s 10.8.0.0/24 -j ACCEPT || true
iptables -I FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT || true

cat > /etc/systemd/system/openvpn-iptables.service <<EOF
[Unit]
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/sbin/iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
ExecStart=/sbin/iptables -I INPUT -p $PROTOCOL --dport $PORT -j ACCEPT
ExecStart=/sbin/iptables -I FORWARD -s 10.8.0.0/24 -j ACCEPT
ExecStart=/sbin/iptables -I FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now openvpn-iptables.service
systemctl enable --now openvpn-server@server.service

mkdir -p /root/openvpn-clients

cat > /root/openvpn-clients/"$CLIENT".ovpn <<EOF
client
dev tun
proto $PROTOCOL
remote $PUBLIC_IP $PORT
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
auth SHA256
verb 3
key-direction 1

<ca>
$(cat /etc/openvpn/server/ca.crt)
</ca>

<cert>
$(sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p' /etc/openvpn/server/easy-rsa/pki/issued/$CLIENT.crt)
</cert>

<key>
$(cat /etc/openvpn/server/easy-rsa/pki/private/$CLIENT.key)
</key>

<tls-auth>
$(cat /etc/openvpn/server/ta.key)
</tls-auth>
EOF

cp /root/openvpn-clients/"$CLIENT".ovpn /home/azureuser/"$CLIENT".ovpn 2>/dev/null || true

echo
echo "=============================================="
echo " OpenVPN installation complete"
echo "=============================================="
echo
echo "Server bind IP:"
echo "  $PRIVATE_IP"
echo
echo "Client public IP:"
echo "  $PUBLIC_IP"
echo
echo "Client config:"
echo "  /root/openvpn-clients/$CLIENT.ovpn"
echo "  /home/azureuser/$CLIENT.ovpn"
echo
echo "Verify remote line:"
echo "  grep '^remote' /home/azureuser/$CLIENT.ovpn"
echo
echo "Azure NSG must allow inbound:"
echo "  $PROTOCOL $PORT"
echo
