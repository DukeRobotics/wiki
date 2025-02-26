# DHCP Server

## Introduction
The Dynamic Host Configuration Protocol (DHCP) server is a service that automatically assigns IP addresses to devices on a local network. This ensures each device has a unique IP address without manual intervention. The DHCP server is essential for the robot to communicate with other devices on the local network formed by the devices connected to the robot's ethernet port.

The robot acts as the DHCP server for the local network. As the server, the robot itself is assigned a static IP address, and it assigns IP addresses to other devices on the network.

The sections below describe how to set up the DHCP server on the robot. The instructions were tested on Ubuntu 24.04.

> [!WARNING]
> Changing the network configuration on Linux can be very tricky and error-prone. Make sure you have plenty of time to troubleshoot any issues that may arise.

> [!TIP]
> If you are connected to the robot via a network (e.g., SSH), you may lose connection when changing the network configuration. It is strongly recommended to have a direct connection to the robot (e.g., HDMI monitor and USB keyboard) to perform these steps.

## Installation
The DHCP server is provided by the `isc-dhcp-server` package. To install the package, run the following command:
```bash
sudo apt install isc-dhcp-server
```
> [!NOTE]
> The Internet Systems Consortium (ISC) stopped maintaining the `isc-dhcp-server` package at the end of 2022. However, it is still supported by Ubuntu 24.04 and provides an easy, reliable way to set up a DHCP server.

## Finding the Network Interface
The first step is to find the network interface that will be used for the DHCP server. The server will assign IP addresses to devices connected to it via this interface.

Typically, this is the ethernet port on the robot. To find the network interface, run the following command:
```bash
ifconfig
```
Look for the interface that corresponds to the ethernet port (e.g., `enp2s0`, `eth0`). This interface will be used in the configuration files.

## Configuration
There are four files that need to be configured to set up the DHCP server:
1. `/etc/dhcp/dhcpd.conf`
2. `/etc/default/isc-dhcp-server`
3. `/etc/NetworkManager/NetworkManager.conf`
4. `/etc/network/interfaces`

The following sections describe the configuration of each file. They show an example configuration of the `enp2s0` interface where the robot has a static IP of `192.168.1.1` and the DHCP server assigns IP addresses in the range `192.168.1.0/24`.

### `/etc/dhcp/dhcpd.conf`
This file contains the configuration for the DHCP server. The following is an example configuration:
```conf
# Specifies the DNS server that clients should use for name resolution. This is the robot's IP address.
# Even though the robot is not a DNS server, it can act as a DNS forwarder and resolve domain names using an external DNS server.
option domain-name-servers 192.168.1.1;

# Sets the default lease time for an IP address (in seconds) if the client does not request a specific lease duration.
default-lease-time 3600;  # 1 hour

# Defines the maximum lease time for an IP address (in seconds), even if the client requests a longer duration.
max-lease-time 7200;  # 2 hours

# Disables dynamic DNS updates. This is appropriate for small networks.
ddns-update-style none;

# Declares that this DHCP server is authoritative for the specified subnet. It will respond to clients even if they were previously configured with a different DHCP server.
authoritative;

# Defines the subnet and its associated DHCP settings.
# The netmask defines which bits of the IP address are fixed (network part) and which bits can vary (host part).
# In this case, the first 24 bits are fixed (192.168.1) and the last 8 bits can vary (0-255).
# Thus, this allows the DHCP server to manage IP addresses within the 192.168.1.x range.
subnet 192.168.1.0 netmask 255.255.255.0 {

    # Specifies the default gateway (router) that clients should use to access other networks.
    # This is the robot's IP address.
    option routers 192.168.1.1;

    # Informs clients about the subnet mask for the network.
    option subnet-mask 255.255.255.0;

    # Defines the range of IP addresses the DHCP server can assign to clients within this subnet.
    # This range should not include the IP address of the robot itself, nor the highest IP address in the subnet (broadcast address; 192.168.1.255 in this case).
    range 192.168.1.2 192.168.1.254;
}
```

### `/etc/default/isc-dhcp-server`
This file contains the configuration for the DHCP server service. It can be left with the default settings. You do not need to specify the interfaces here because they are defined in `/etc/network/interfaces`.
```conf
# Defaults for isc-dhcp-server (sourced by /etc/init.d/isc-dhcp-server)

# Path to dhcpd's config file (default: /etc/dhcp/dhcpd.conf).
#DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
#DHCPDv6_CONF=/etc/dhcp/dhcpd6.conf

# Path to dhcpd's PID file (default: /var/run/dhcpd.pid).
#DHCPDv4_PID=/var/run/dhcpd.pid
#DHCPDv6_PID=/var/run/dhcpd6.pid

# Additional options to start dhcpd with.
#	Don't use options -cf or -pf here; use DHCPD_CONF/ DHCPD_PID instead
#OPTIONS=""

# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
#	Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACESv4=""
INTERFACESv6=""
```

### `/etc/NetworkManager/NetworkManager.conf`
This file contains the configuration for NetworkManager, which manages network connections on Ubuntu.
```conf
[main]
plugins=ifupdown,keyfile

[ifupdown]
managed=true

[device]
wifi.scan-rand-mac-address=no
```
The most important lines in this file are:
- `plugins=ifupdown,keyfile`: Specifies the plugins that NetworkManager should use. Must include `ifupdown` for managing network interfaces defined in `/etc/network/interfaces`.
- `managed=true`: Tells NetworkManager to manage network interfaces defined in `/etc/network/interfaces`. This is necessary for the DHCP server to work correctly.

### `/etc/network/interfaces`
This file configures the network interfaces.
```conf
# Loopback interface, used for internal system communication
# Do not modify this section
auto lo
iface lo inet loopback

# Automatically bring up the interface used for the DHCP server when the system boots (enp2s0 in this case).
auto enp2s0

# Configure the enp2s0 interface with a static IP address.
iface enp2s0 inet static

# Set the static IP address for the interface. This should be the same as the domain-name-servers and routers options in dhcpd.conf.
address 192.168.1.1

# Define the subnet mask for the network. This should match the subnet mask in dhcpd.conf.
netmask 255.255.255.0
```

## Enabling the DHCP Server
After configuring the files, you need to enable the DHCP server service. Run the following commands:
```bash
sudo systemctl enable isc-dhcp-server
```

Also, enable NetworkManager to manage the network interfaces:
```bash
sudo systemctl enable NetworkManager
```

Finally, reboot the robot to apply the changes:
```bash
sudo reboot
```

After the robot boots up, it may take up to two minutes for the DHCP server to start and for devices connected to the local network to receive IP addresses.

To see if the DHCP server is running, you can check the status using the following command:
```bash
sudo systemctl status isc-dhcp-server
```

## Change the IP Address
To change the IP address of the robot or the IP address range assigned by the DHCP server, you need to modify the following files:
- `/etc/dhcp/dhcpd.conf`: Change the IP addresses in the `domain-name-servers`, `routers`, `range`, and `subnet` fields.
- `/etc/network/interfaces`: Change the IP address in the `address` field.