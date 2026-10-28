# Οδηγός: OpenVPN VPN layer 2 (TAP) και VPN layer 3 (TUN), γεφυρωμένα

## Τι θα κάνουμε

- Έχουμε ένα TAP VPN για να παίζουμε παιχνίδια.
- Έχουμε ένα TUN VPN για να συνδέονται κινητά Android.
- Ο TAP και ο TUN εξυπηρετητής είναι η ίδια συσκευή.
- Οι συσκευές που ανήκουν στο TAP επικοινωνούν με τις συσκευές που ανήκουν στο TUN VPN και αντίστροφα.

## Γιατί TAP

Το Layer 2 είναι το επίπεδο της διεύθυνσης MAC.

Δεν υπάρχει άλλος λόγος πέρα από το να παίζεις LAN παιχνίδια με φίλους σε όλο τον κόσμο.

Δυστυχώς, με Layer 3 δεν γίνεται broadcast στο `x.x.x.255`.

Το Layer 3 είναι το επίπεδο IP. Τώρα, γιατί δεν γίνεται multicast σε επίπεδο IP, μόνο το OpenVPN ξέρει.

## Τα δίκτυα

Ο OpenVPN TAP και TUN εξυπηρετητής είναι η ίδια συσκευή.

Ο OpenVPN εξυπηρετητής βρίσκεται στο φυσικό δίκτυο `10.0.0.0/24` και έχει διεύθυνση `10.0.0.2`.

Όλες οι συσκευές που ανήκουν στο ίδιο φυσικό δίκτυο με τον OpenVPN εξυπηρετητή (`10.0.0.0/24`) **θα είναι προσβάσιμες από το TAP VPN χωρίς** να χρειάζεται να τρέχουν OpenVPN TAP πελάτη (ακόμα και τυχόν τηλεοράσεις, δικτυακοί εκτυπωτές, διακόπτες φωτισμού κ.λπ.).

Όλες οι συσκευές που ανήκουν στο ίδιο φυσικό δίκτυο με τον OpenVPN εξυπηρετητή (`10.0.0.0/24`), αν δεν εκτελούν OpenVPN TUN πελάτη, **δεν θα είναι προσβάσιμες από το TUN VPN**. Αν θέλουμε να είναι προσβάσιμες χωρίς OpenVPN TUN πελάτη, εφαρμόζουμε τη διαδικασία που αναφέρεται παρακάτω.

Όλες οι απομακρυσμένες συσκευές (εκτός φυσικού δικτύου `10.0.0.0/24`) πρέπει να εκτελούν OpenVPN client.

Όλες οι απομακρυσμένες συσκευές (εκτός φυσικού δικτύου `10.0.0.0/24`), αν ανήκουν σε κάποιο φυσικό δίκτυο, αυτό δεν θα πρέπει να είναι το `10.0.0.0/24` ούτε το `10.8.0.0/24`. Ένα αποδεκτό φυσικό δίκτυο είναι το `192.168.1.0/24`.

Όλες οι απομακρυσμένες συσκευές (εκτός φυσικού δικτύου `10.0.0.0/24`), μετά τη σύνδεση με το TAP VPN, θα αποκτήσουν μια επιπλέον διεύθυνση IP στο εικονικό τοπικό δίκτυο `10.0.0.0/24`, πέρα από τη διεύθυνση του φυσικού τοπικού τους δικτύου (`192.168.1.0/24`).

Όλες οι συσκευές που εκτελούν VPN TUN πελάτη, μετά τη σύνδεση με το TUN VPN, θα αποκτήσουν μια επιπλέον διεύθυνση IP στο εικονικό τοπικό δίκτυο `10.8.0.0/24`, πέρα από τη διεύθυνση του φυσικού τοπικού τους δικτύου (`192.168.1.0/24`).

## Για να συνδεθείς στο OpenVPN

Κάθε πελάτης και ο εξυπηρετητής έχουν ένα πιστοποιητικό ασφαλείας υπογεγραμμένο από μια αρχή πιστοποίησης.

Επίσης, έχουν από ένα αρχείο ρυθμίσεων, το οποίο πρέπει να ταιριάζει μεταξύ τους σε πολλές σημαντικές παραμέτρους που σχετίζονται με τη συνδιαλλαγή.

## ΠΡΟΣΟΧΗ!

Χρησιμοποιήστε την τελευταία έκδοση του OpenSSL για τη δημιουργία των πιστοποιητικών που περιγράφεται παρακάτω.

Κατά προτίμηση, κατεβάστε και χρησιμοποιήστε το OpenSSL για Windows, γιατί το Linux (ειδικά το Debian για επεξεργαστές ARM) συχνά διαθέτει παλαιότερη έκδοση.

Ο λόγος είναι ότι νεότερο OpenSSL μπορεί να απορρίπτει πιστοποιητικά που δημιουργήθηκαν με παλαιότερο OpenSSL, για κάποιον λόγο ασφαλείας. (Μου έχει συμβεί.)

## Δημιουργώντας το πιστοποιητικό της αρχής πιστοποίησης

Για να δημιουργηθεί το πιστοποιητικό της αρχής πιστοποίησης, εκτελούμε τις εντολές:

```bash
openssl genrsa -des3 -out ca.key 4096
openssl req -new -x509 -utf8 -days 36500 -key ca.key -out ca.crt
```

Προκύπτουν το δημόσιο πιστοποιητικό `ca.crt` και το ιδιωτικό κλειδί `ca.key`.

## Δημιουργώντας το πιστοποιητικό του εξυπηρετητή

Απαιτείται η δημιουργία ενός αρχείου `openssl.x509.server.conf` με τα εξής περιεχόμενα:

```ini
# These extensions are added when 'ca' signs a request.
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
```

Για να δημιουργηθεί το πιστοποιητικό του εξυπηρετητή, εκτελούμε τις εντολές:

```bash
openssl genrsa -out server.key 4096
openssl req -new -utf8 -key server.key -out server.csr
openssl x509 -req -days 36500 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt -extfile openssl.x509.server.conf
rm server.csr
```

Προκύπτουν το δημόσιο πιστοποιητικό `server.crt` και το ιδιωτικό κλειδί `server.key`.

**ΠΡΟΣΟΧΗ!** Το OpenSSL δεν απαιτεί να συμπληρωθεί κάποιο Common Name (CN) στα στοιχεία του πιστοποιητικού. Το OpenVPN, όμως, το απαιτεί σύμφωνα με αυτήν τη διαδικασία. Συμπληρώστε το υποχρεωτικά.

## Δημιουργώντας ένα πιστοποιητικό για κάθε πελάτη

Απαιτείται η δημιουργία ενός αρχείου `openssl.x509.client.conf` με τα εξής περιεχόμενα:

```ini
# These extensions are added when 'ca' signs a request.
keyUsage = digitalSignature
extendedKeyUsage = clientAuth
```

Για να δημιουργηθεί το πιστοποιητικό κάθε πελάτη, εκτελούμε για κάθε πελάτη τις εντολές:

```bash
openssl genrsa -out client.key 4096
openssl req -new -utf8 -key client.key -out client.csr
openssl x509 -req -days 36500 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out client.crt -extfile openssl.x509.client.conf
rm client.csr
```

Προκύπτουν το δημόσιο πιστοποιητικό `client.crt` και το ιδιωτικό κλειδί `client.key`.

**ΠΡΟΣΟΧΗ!** Το OpenSSL δεν απαιτεί να συμπληρωθεί κάποιο Common Name (CN) στα στοιχεία του πιστοποιητικού. Το OpenVPN, όμως, το απαιτεί σύμφωνα με αυτήν τη διαδικασία. Συμπληρώστε το υποχρεωτικά.

## Δημιουργώντας το κλειδί tls-auth

Για να δημιουργηθεί το κλειδί `tls-auth`, το οποίο παρέχει ένα επιπλέον επίπεδο ασφάλειας, εκτελούμε την εντολή:

```bash
openvpn --genkey --secret ta.key
```

Προκύπτει το ιδιωτικό κλειδί `ta.key`.

## Δημιουργώντας το αρχείο Diffie–Hellman

Για να δημιουργηθεί το αρχείο Diffie–Hellman 4096 bit, εκτελούμε την εντολή:

```bash
openssl dhparam -out dh4096.pem 4096
```

Προκύπτει το αρχείο `dh4096.pem` μετά από αρκετή ώρα — ενδεχομένως ώρες ή ημέρες.

## Στήνοντας τον εξυπηρετητή TAP

Τροποποιούμε το αρχείο ρυθμίσεων δικτύου `/etc/network/interfaces` ως εξής:

```text
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

Απενεργοποιούμε οτιδήποτε έχει σχέση με το `eth0` και ενεργοποιούμε τη γέφυρα δικτύου `br0` με τα ίδια χαρακτηριστικά.

Στο `bridge_ports` δεν μπορούμε να βάλουμε και το `tap0` (τη διασύνδεση δικτύου που δημιουργεί το OpenVPN), επειδή δεν έχει εκτελεστεί ακόμη ο OpenVPN εξυπηρετητής για να τη δημιουργήσει. Η ρύθμιση δικτύου είναι προαπαιτούμενο για την εκκίνηση του OpenVPN εξυπηρετητή.

Μεταφέρουμε τα αρχεία `ca.crt`, `dh4096.pem`, `server.crt`, `server.key` και `ta.key` στον φάκελο `/etc/openvpn`, με ιδιοκτήτη `root:root` και δικαιώματα `400`.

Δημιουργούμε το εκτελέσιμο αρχείο `/etc/openvpn/openvpn_up` με τα εξής περιεχόμενα:

```bash
#!/bin/bash
brctl addif br0 "$1"
ifconfig "$1" up
```

Το αρχείο αυτό, μόλις ολοκληρωθεί η δημιουργία της διασύνδεσης δικτύου `tap0` (`$1`) από το OpenVPN, συνδέει τη διασύνδεση με τη γέφυρα δικτύου `br0`. Ακολούθως, ενεργοποιεί τη διασύνδεση `tap0`.

Σε νεότερα συστήματα, προκειμένου να αποφύγουμε την εγκατάσταση του `net-tools` (για το `ifconfig`), χρησιμοποιούμε την εντολή `ip` του βασικού πακέτου `iproute2`. Το παραπάνω script γράφεται τότε ως εξής:

```bash
#!/bin/bash
ip link set "$1" master br0
ip link set "$1" up
```

Ρυθμίζουμε ή δημιουργούμε το αρχείο `/etc/openvpn/server_tap.conf` ως εξής (τα σχόλια αφαιρέθηκαν):

```ini
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
;keepalive 10000 11000
tls-auth ta.key 0          # This file is secret
cipher AES-256-CBC
;compress lz4-v2           # Maybe I add it later
;push "compress lz4-v2"
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
script-security 2
up openvpn_up
;down openvpn_down
```

**Παρατήρηση:** Σε όλους τους VPN TAP πελάτες γνωστοποιούμε το VPN TUN δίκτυο `10.8.0.0`.

Κάνουμε επανεκκίνηση.

## Στήνοντας τον εξυπηρετητή TUN

Χρησιμοποιούμε τα ίδια αρχεία `ca.crt`, `dh4096.pem`, `server.crt`, `server.key` και `ta.key` με τον εξυπηρετητή TAP, οπότε δεν αλλάζουμε κάτι σε αυτά.

Ρυθμίζουμε ή δημιουργούμε το αρχείο `/etc/openvpn/server_tun.conf` ως εξής (τα σχόλια αφαιρέθηκαν):

```ini
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
tls-auth ta.key 0          # This file is secret
cipher AES-256-CBC
;compress lz4-v2           # Maybe I add it later
;push "compress lz4-v2"
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

Παρατηρήσεις:

- Χρησιμοποιούμε την επόμενη θύρα για το TUN VPN: `1195`.
- Δεν εκτελούμε το script που θα γεφύρωνε το `tun0` με τα `tap0` και `eth0` στο `br0`, επειδή το `tun0` λειτουργεί στο Layer 3 και δεν γεφυρώνεται με αυτόν τον τρόπο.
- Σε όλους τους VPN TUN πελάτες γνωστοποιούμε το VPN TAP δίκτυο `10.0.0.0`.

Όπως αναφέρθηκε, οι συσκευές του ίδιου φυσικού δικτύου με τον εξυπηρετητή δεν εκτελούν VPN TAP πελάτη, οπότε δεν γνωρίζουν καθόλου για το δίκτυο TUN (`10.8.0.0/24`).

Προκειμένου να φτάσουν στον προορισμό τους τα πακέτα δεδομένων που έρχονται στον εξυπηρετητή από τις παραπάνω συσκευές με προορισμό κάποια συσκευή του VPN TUN δικτύου (`10.8.0.0/24`), και αντίστροφα, πρέπει να είναι ενεργό το IP forwarding.

Για να ελέγξουμε αν είναι ενεργό, εκτελούμε την εντολή:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

Αν λάβουμε `1` (ναι), όλα καλά. Αν λάβουμε `0` (όχι), πρέπει να το ενεργοποιήσουμε.

Επεξεργαζόμαστε το αρχείο `/etc/sysctl.conf` και αφαιρούμε τον σχολιασμό από τη γραμμή:

```ini
net.ipv4.ip_forward=1
```

Για να ενημερωθεί το σύστημα για την αλλαγή, εκτελούμε την εντολή:

```bash
sysctl -p
```

Από το Debian 13 και μετά, μπορούμε να δημιουργήσουμε ξεχωριστό αρχείο και να ενημερώσουμε το σύστημα:

```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/90-ip-forward.conf
sysctl --system
```

Κάνουμε επανεκκίνηση.

## Στήνοντας τον πελάτη TAP

Μεταφέρουμε τα αρχεία `ca.crt`, `dh4096.pem`, `client.crt`, `client.key` και `ta.key` στον φάκελο `/etc/openvpn`, με ιδιοκτήτη `root:root` και δικαιώματα `400`.

Αν ο πελάτης εκτελεί Windows, ακολουθούμε την αντίστοιχη διαδικασία.

Ρυθμίζουμε ή δημιουργούμε το αρχείο `/etc/openvpn/client.conf` ως εξής (τα σχόλια αφαιρέθηκαν):

```ini
remote myserver.myftp.org 1194
client
proto udp
dev tap
ca ca.crt
cert client.crt
key client.key          # This file is secret
tls-auth ta.key 1        # This file is secret
cipher AES-256-CBC
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

Εναλλακτικά, μπορούμε να ενσωματώσουμε όλα τα πιστοποιητικά και τα κλειδιά στο αρχείο ρυθμίσεων, το οποίο θα έχει περίπου την εξής μορφή:

```ini
remote myserver.myftp.org 1194
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
...key contents...
-----END RSA PRIVATE KEY-----
</key>
;keepalive 60 120
key-direction 1        # Complementary of tls-auth
<tls-auth>
-----BEGIN OpenVPN Static key V1-----
...key contents...
-----END OpenVPN Static key V1-----
</tls-auth>
cipher AES-256-CBC
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

Εκτελούμε τον πελάτη OpenVPN.

## Στήνοντας τον πελάτη TUN

Μεταφέρουμε τα αρχεία `ca.crt`, `dh4096.pem`, `client.crt`, `client.key` και `ta.key` στον φάκελο `/etc/openvpn`, με ιδιοκτήτη `root:root` και δικαιώματα `400`.

Αν ο πελάτης εκτελεί Windows, ακολουθούμε την αντίστοιχη διαδικασία.

Ρυθμίζουμε ή δημιουργούμε το αρχείο `/etc/openvpn/client.conf` ως εξής (τα σχόλια αφαιρέθηκαν):

```ini
remote myserver.myftp.org 1195
client
proto udp
dev tun
ca ca.crt
cert client.crt
key client.key          # This file is secret
tls-auth ta.key 1        # This file is secret
cipher AES-256-CBC
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

Εναλλακτικά, μπορούμε να ενσωματώσουμε όλα τα πιστοποιητικά και τα κλειδιά στο αρχείο ρυθμίσεων, με την ίδια λογική που ακολουθήσαμε και για το TAP.

Εκτελούμε τον πελάτη OpenVPN.

## Συνδέοντας συσκευές του ίδιου φυσικού δικτύου με τον εξυπηρετητή, που δεν εκτελούν TAP ή TUN πελάτη, με το VPN TUN

Αν η συσκευή εκτελεί Windows, εκτελούμε την εντολή:

```cmd
route -p add 10.8.0.0 mask 255.255.255.0 10.0.0.2
```

Αν η συσκευή εκτελεί Linux, εκτελούμε μια αντίστοιχη εντολή.
'''
