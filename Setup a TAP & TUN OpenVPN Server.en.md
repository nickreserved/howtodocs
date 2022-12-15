Guide: Setup OpenVPN layer 2 (TAP) and layer 3 (TUN), bridged
=============================================================


What we will do
---------------
- Setup a TAP VPN to play LAN games.
- Setup a TUN VPN to connect android phones.
- TAP and TUN server is the same machine.
- Machines in TAP VPN communicate with machines in TUN VPN and vice-versa.


Why TAP
-------
Layer 2 is MAC address level.

There is no other way to play LAN games with your friends in the world.

Unfortunately with layer 3 there is no multicast in `x.x.x.255`.

Layer 3 is IP address level.


Networks
--------
TAP and TUN server is the same machine.

OpenVPN server belongs to LAN `10.0.0.0/24` and has address `10.0.0.2`.

All machines in the same LAN with OpenVPN server (`10.0.0.0/24`) **can** be accessible from TAP VPN **without** the need to run OpenVPN TAP client (TVs, printers, smart light switches, etc included).

All machines in the same LAN with OpenVPN server (`10.0.0.0/24`), if not run OpenVPN TUN client, **cannot** be accessible from TUN VPN. If we want be accessible without OpenVPN TUN client, we do the procedure at the end of article.

All machines **not** in the same LAN with OpenVPN server (`10.0.0.0/24`) must run OpenVPN client.

All machines **not** in the same LAN with OpenVPN server (`10.0.0.0/24`), if they belong in a LAN, this LAN must not be `10.0.0.0/24` or `10.8.0.0/24`. An acceptable LAN can be `192.168.1.0/24`.

All machines **not** in the same LAN with OpenVPN server (`10.0.0.0/24`), after TAP VPN connection, have an extra IP address `10.0.0.0/24` in TAP VPN, keeping their original LAN address `192.168.1.0/24`.

All machines which run VPN TUN client, after TUN VPN connection, have an extra IP address `10.8.0.0/24` in TUN VPN, keeping their original LAN address `192.168.1.0/24`.


Install of required packages
----------------------------
We install the required packages. Installation refer to Debian or Debian derivatives.
```Shell
apt install openssl openvpn bridge-utils net-tools
```


To be connected in OpenVPN
--------------------------
Every client and the server have a certification signed from a certification authority.

Also they have a configuration file which all of them must have many settings the same, or else connection will not be established.


Warning about OpenSSL!
----------------------
Use the latest version of OpenSSL to create certifications (procedures follow).

Download and use OpenSSL for Windows because Linux (specially Debian for ARM CPUs) has older OpenSSL version.

Newer OpenSSL can reject certificates issued with older OpenSSL for some security reason (I suffer from this).


Creating certification authority
--------------------------------
To create certification of certification authority, you must run the commands:
```Shell
openssl genrsa -des3 -out ca.key 4096
openssl req -new -x509 -utf8 -days 36500 -key ca.key -out ca.crt
```
and we have public `ca.crt` and private key `ca.key`.


Creating server's certification
-------------------------------
We must create a file `openssl.x509.server.conf` with contents:
```Shell
# These extensions are added when 'ca' signs a request.
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage=serverAuth
```

To create certification of server, you must run the commands:
```Shell
openssl genrsa -out server.key 4096
openssl req -new -utf8 -key server.key -out server.csr
openssl x509 -req -days 36500 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt -extfile openssl.x509.server.conf
rm server.csr
```
and we have public `server.crt` and private key `server.key`.

**Warning!** OpenSSL does not require Common Name (CA) for certificate but OpenVPN requires it. So, you **must** give a Common Name!


Creating each client's certification
------------------------------------
We must create a file `openssl.x509.client.conf` with contents:
```Shell
# These extensions are added when 'ca' signs a request.
keyUsage = digitalSignature
extendedKeyUsage=clientAuth
```

To create certification of each client, you must run the commands for every client:
```Shell
openssl genrsa -out client.key 4096
openssl req -new -utf8 -key client.key -out client.csr
openssl x509 -req -days 36500 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out client.crt -extfile openssl.x509.client.conf
rm client.csr
```
and we have public `client.crt` and private key `client.key`.

**Warning!** OpenSSL does not require Common Name (CA) for certificate but OpenVPN requires it. So, you **must** give a Common Name!


Creating the tls-auth key
-------------------------
To create tls-auth key, which provides more security, you must run the commands:
```Shell
openvpn --genkey --secret ta.key
```
and we have the private key `ta.key`.


Creating Diffie-Hellman file
----------------------------
To create Diffie-Hellman file at 4096 bits, you must run the commands:
```Shell
openssl dhparam -out dh4096.pem 4096
```
and we have -after **a lot** of time- file `dh4096.pem`.


Setting up the TAP server
-------------------------
We modify the network configuration file `/etc/network/interfaces` as folling:
```INI
#auto eth0
##iface eth0 inet dhcp
#iface eth0 inet static
#address 10.0.0.2
#netmask 255.255.255.0
#gateway 10.0.0.1
#dns-nameservers 10.0.0.1

auto br0
iface br0 inet static
address 10.0.0.2
netmask 255.255.255.0
gateway 10.0.0.1
network 10.0.0.0
broadcast 10.0.0.255
bridge_ports eth0
```
Disable everything related with `eth0` and enable network bridge `br0` with the same options.

On option `bridge_ports` we don't put `tap0` (`tap0` is the network interface created from OpenVPN) because OpenVPN hasn't executed yet: Network must be setup before OpenVPN.

Copy files `ca.crt`, `dh4096.pem`, `server.crt`, `server.key, `ta.key` to folder `/etc/openvpn`, with ownership `root:root` and permissions `400`.

Create an **executable** file `/etc/openvpn/openvpn_up` with contents:
```Shell
#!/bin/bash
brctl addif br0 $1
ifconfig $1 up
```
This file, when OpenVPN create network interface `tap0` (`$1`), connects this network interface under the bridge `br0`. Then up network interface `tap0`.

Modify or create the file `/etc/openvpn/server_tap.conf` with following contents (comments stripped):
```INI
port 1194
proto udp
dev tap
ca ca.crt
cert server.crt
key server.key             # This file is secret
dh dh4096.pem
topology subnet
ifconfig-pool-persist ipp_tap.txt
server-bridge 10.0.0.2 255.255.255.0 10.0.0.192 10.0.0.254
push "route 10.8.0.0 255.255.255.0"
client-to-client
;keepalive 10000 11000      # super max or disable or else disconnections
tls-auth ta.key 0           # This file is secret
cipher AES-256-CBC
;compress lz4-v2			# Maybe I add it later
;push "compress lz4-v2"
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
script-security 2
up openvpn_up
;down openvpn_down          # no need
```

Comments:
- OpenVPN TAP server informs all OpenVPN TAP clients for VPN TUN network `10.8.0.0`.

Reboot.


Setting up the TUN server
-------------------------
We use the same files `ca.crt`, `dh4096.pem, `server.crt, `server.key, `ta.key` with TAP server, so no changes.

Modify or create the file `/etc/openvpn/server_tun.conf` with following contents (comments stripped):
```INI
port 1195
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key             # This file is secret
dh dh4096.pem
topology subnet
ifconfig-pool-persist ipp_tun.txt
server 10.8.0.0 255.255.255.0
push "route 10.0.0.0 255.255.255.0"
client-to-client
;keepalive 10000 11000
tls-auth ta.key 0           # This file is secret
cipher AES-256-CBC
;compress lz4-v2			# Maybe I add it later
;push "compress lz4-v2"
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

Comments:
- We use the next port for TUN VPN `1195`.
- We do not run executable file which bridges to `br0` the `tun0` with `tap0` and `eth0`, because `tun0` is layer 3 and cannot bridged.
- OpenVPN TUN server informs all OpenVPN TUN clients for VPN TAP network `10.0.0.0`.

As I say before, machines in the same LAN with server, do not run VPN TAP client, so they do not know about TUN network (`10.8.0.0/24`).

In order, for packages, to arrive in their destination, packages which arrive on server from machines above, with destination a machine of VPN TUN network (`10.8.0.0/24`) and vice versa, IP forwarding must be enabled on server.

We check IP forwarding with command:
```Shell
cat /proc/sys/net/ipv4/ip_forward
```
If result is 1 (yes) ok. If result is 0 (no) we must enable IP forwarding.

Edit file `/etc/sysctl.conf` and uncomment line:
```INI
net.ipv4.ip_forward=1
```
save file and inform system for change with command:
```Shell
sysctl -p
```
Reboot.


Setting up the TAP client
-------------------------
Copy files `ca.crt`, `dh4096.pem, `client.crt, `client.key`, `ta.key`, to folder `/etc/openvpn`, with ownership `root:root` and permissions `400`.

If we have windows in client, do something similar. It is easier.

Modify or create the file `/etc/openvpn/client.conf` with following contents (comments stripped):
```INI
remote mydomain.myftp.org 1194
client
proto udp
dev tap
ca ca.crt
cert client.crt
key client.key         # This file is secret
tls-auth ta.key 1       # This file is secret
cipher AES-256-CBC
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

Or else we can insert inline the certificates inside configuration file, so contents are:
```INI
remote mydomain.myftp.org 1194
client
proto udp
dev tap
<ca>
-----BEGIN CERTIFICATE-----
...certificate contents...
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
...certificate contents...
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN RSA PRIVATE KEY-----
...certificate contents...
-----END RSA PRIVATE KEY-----
</key>
;keepalive 60 120
key-direction 1        # Complementary of tls-auth
<tls-auth>
-----BEGIN OpenVPN Static key V1-----
...certificate contents...
-----END OpenVPN Static key V1-----
</tls-auth>
cipher AES-256-CBC
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

Run the OpenVPN client service.


Setting up the TUN client
-------------------------
Copy files `ca.crt`, `dh4096.pem`, `client.crt, `client.key`, `ta.key` to folder `/etc/openvpn`, with ownership `root:root` and permissions `400`.
If we have windows in client, do something similar. It is easier.

Modify or create the file `/etc/openvpn/client.conf` with following contents (comments stripped):
```INI
remote myserver.myftp.org 1195
client
proto udp
dev tun
ca ca.crt
cert client.crt
key client.key         # This file is secret
tls-auth ta.key 1       # This file is secret
cipher AES-256-CBC
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

Or else we can insert inline the certificates inside configuration file, as we do for TAP client.

Run the OpenVPN client service.


Connecting machines in the same LAN with server, which does not run TAP or TUN client, with VPN TUN
---------------------------------------------------------------------------------------------------
If machine runs Windows, execute the following command:
```Shell
route -p add 10.8.0.0 mask 255.255.255.0 10.0.0.2
```
If machine runs Linux, something similar.
