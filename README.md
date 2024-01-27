# VPN tunnel from ubuntu to Fortigate
## Ipsec.conf changes - Add thie connection settings
conn to-fortigate
	left=%defaultroute
	leftauth=psk
	leftid=yourlocalIP
	right=publicipofgate
	rightid=publicipofgate
	rightauth=psk
	forceencaps=yes
	ike=aes256-sha1-modp1024!
	esp=aes256-sha1!
	auto=start
	dpdaction=restart
	dpddelay=30s
	dpdtimeout=60s
	type=tunnel
	mark=10
	leftsubnet=0.0.0.0/0
	rightsubnet=0.0.0.0/0
	leftupdown=/etc/strongswan/vpn-up.sh

## Modify /etc/strongswan.conf to not install routes - this is because we are using route-based VPNs
charon {
        load_modular = yes
        install_routes = no <=
        install_virtual_ip = no <=
        plugins {
                include strongswan.d/charon/*.conf
                agent {
                        load = no
                }
                kernel-netlink {
                        port_nat_t = 4500
                }       
        }
}

## Create ipsec secrets
append the psk info to the file /etc/ipsec.secrets
yourip	remoteip : PSK extrastrongsecret


## Create interface creation script
/etc/strongswan/vpn-up.sh
```bash
#!/bin/bash

# Configuration variables
UBUNTU_IP="yourlocalip"
FORTIGATE_IP="publicip-of-fortigate"
VTI_KEY="10"
VPN_LOCAL_IP="192.168.121.34/30"
VTI_IF="vpn0"

# Check if the VTI interface already exists
if ip link show $VTI_IF &> /dev/null; then
    echo "VTI interface $VTI_IF already exists, deleting it first."
    ip link del $VTI_IF
fi

# Commands to set up the VTI interface
ip link add name $VTI_IF type vti local $UBUNTU_IP remote $FORTIGATE_IP key $VTI_KEY
ip addr add $VPN_LOCAL_IP dev $VTI_IF
ip link set up dev $VTI_IF
ip link set dev $VTI_IF mtu 1380

# Add a route for the VPN
ip route add 192.168.121.33/32 dev $VTI_IF
ip route add 192.168.0.0/24 via 192.168.121.33
ip route add 192.168.0.0/16 via 192.168.121.33
ip route add 172.16.0.0/12 via 192.168.121.33
ip route add 10.0.0.0/8 via 192.168.121.33
```
## Permissions on file
chmod +x /etc/strongswan/vpn-up.sh

# Finally bring up the tunnel
systemctl restart strongswan
