Οδηγός: OpenVPN VPN layer 2 (TAP) και VPN layer 3 (TUN), γεφυρωμένα
===================================================================


Τι θα κάνουμε
-------------
- Έχουμε ένα TAP VPN για να παίζουμε παιχνίδια.
- Έχουμε ένα TUN VPN για να συνδέονται κινητά android.
- O TAP και ο TUN εξυπηρετητής είναι η ίδια συσκευή.
- Οι συσκευές που ανήκουν στο TAP επικοινωνούν με τις συσκευές που ανήκουν στο TUN VPN και αντίστροφα.



Γιατί TAP
---------
Layer 2 είναι επίπεδο MAC διεύθυνσης.

Δεν υπάρχει άλλος λόγος πέρα από το να παίζεις LAN παιχνίδια με φίλους σε όλο τον κόσμο.

Δυστυχώς με layer 3 δεν γίνεται broadcast στο `x.x.x.255`.

Layer 3 είναι επίπεδο IP.


Τα δίκτυα
---------
Ο OpenVPN TAP και TUN εξυπηρετητής είναι η ίδια συσκευή.

Ο OpenVPN εξυπηρετητής βρίσκεται στο φυσικό δίκτυο `10.0.0.0/24` και έχει διεύθυνση `10.0.0.2`.

Όλες οι συσκευές που ανήκουν στο ίδιο φυσικό δίκτυο με τον OpenVPN εξυπηρετητή (`10.0.0.0/24`) ΘΑ είναι προσβάσιμες από το TAP VPN **χωρίς** να χρειάζεται να τρέχουν OpenVPN TAP πελάτη (ακόμα και τυχόν τηλεοράσεις, δικτυακοί εκτυπωτές, διακόπτες φωτισμού κτλ).

Όλες οι συσκευές που ανήκουν στο ίδιο φυσικό δίκτυο με τον OpenVPN εξυπηρετητή (`10.0.0.0/24`), αν δεν εκτελούν OpenVPN TUN πελάτη, **δεν θα** είναι προσβάσιμες από το TUN VPN. Αν θέλουμε να είναι προσβάσιμες χωρίς OpenVPN TUN πελάτη, εφαρμόζουμε τη διαδικασία που αναφέρεται παρακάτω.

Όλες οι απομακρυσμένες συσκευές (εκτός φυσικού δικτύου `10.0.0.0/24`) πρέπει να εκτελούν OpenVPN client.

Όλες οι απομακρυσμένες συσκευές (εκτός φυσικού δικτύου `10.0.0.0/24`), αν ανήκουν σε κάποιο φυσικό δίκτυο, αυτό δεν θα πρέπει να είναι το `10.0.0.0/24` και το `10.8.0.0/24`. Ένα αποδεκτό φυσικό δίκτυο είναι το `192.168.1.0/24`.

Όλες οι απομακρυσμένες συσκευές (εκτός φυσικού δικτύου `10.0.0.0/24`), μετά τη σύνδεση με το TAP VPN, θα αποκτήσουν μια επιπλέον διεύθυνση IP στο εικονικό τοπικό δίκτυο `10.0.0.0/24`, πέραν αυτή του φυσικού τοπικού τους δικτύου `192.168.1.0/24`.

Όλες οι συσκευές που εκτελούν VPN TUN πελάτη, μετά τη σύνδεση με το TUN VPN, θα αποκτήσουν μια επιπλέον διεύθυνση IP στο εικονικό τοπικό δίκτυο `10.8.0.0/24`, πέραν αυτή του φυσικού τοπικού τους δικτύου `192.168.1.0/24`.


Εγκατάσταση απαιτούμενων πακέτων
--------------------------------
Εξασφαλίζουμε την εγκατάσταση των απαιτούμενων πακέτων. Η εγκατάσταση απευθύνεται σε Debian ή βασισμένα σε Debian Linux.
```Shell
apt install openssl openvpn bridge-utils net-tools
```


Για να συνδεθείς στο OpenVPN
----------------------------
Κάθε πελάτης και ο εξυπηρετητής έχουν ένα πιστοποιητικό ασφαλείας υπογεγραμμένο από μια αρχή πιστοποίησης

Επίσης έχουν από ένα αρχείο ρυθμίσεων το οποίο πρέπει να ταιριάζει μεταξύ των σε πολλές σημαντικές παραμέτρους που έχουν σχέση με την συνδιαλλαγή.


Προσοχή σχετικά με το OpenSSL!
------------------------------
Χρησιμοποιείστε την τελευταία έκδοση του OpenSSL για τη δημιουργία πιστοποιητικών που ακολουθεί.

Κατά προτίμηση κατεβάστε και χρησιμοποιείστε το OpenSSL για Windows γιατί το Linux (ειδικά το Debian για επεξεργαστές ARM) σχεδόν πάντα έχει παλαιότερη έκδοση.

Αυτό γιατί νεότερο OpenSSL μπορεί να απορρίπτει πιστοποιητικά που δημιουργήθηκαν με παλαιότερο OpenSSL για κάποιο λόγο ασφαλείας (το έχω πάθει).


Δημιουργώντας το πιστοποιητικό της αρχής πιστοποίησης
-----------------------------------------------------
Για να δημιουργηθεί το πιστοποιητικό της αρχής πιστοποίησης εκτελούμε τις εντολές:
```Shell
openssl genrsa -des3 -out ca.key 4096
openssl req -new -x509 -utf8 -days 36500 -key ca.key -out ca.crt
```
και προκύπτει το δημόσιο `ca.crt` και το ιδιωτικό `ca.key` κλειδί.


Δημιουργώντας το πιστοποιητικό του εξυπηρετητή
----------------------------------------------
Εδώ απαιτείται η δημιουργία ενός αρχείου `openssl.x509.server.conf` με περιεχόμενα:
```Shell
# These extensions are added when 'ca' signs a request.
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage=serverAuth
```

Για να δημιουργηθεί το πιστοποιητικό του εξυπηρετητή εκτελούμε τις εντολές:
```Shell
openssl genrsa -out server.key 4096
openssl req -new -utf8 -key server.key -out server.csr
openssl x509 -req -days 36500 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt -extfile openssl.x509.server.conf
rm server.csr
```
και προκύπτει το δημόσιο `server.crt` και το ιδιωτικό `server.key` κλειδί.

**Προσοχή!** Το OpenSSL δεν απαιτεί να συμπληρωθεί κάποιο Common Name (CA) στα στοιχεία του πιστοποιητικού. Το OpenVPN όμως το απαιτεί. Οπότε συμπληρώστε το υποχρεωτικά!


Δημιουργώντας ένα πιστοποιητικό για κάθε πελάτη
-----------------------------------------------
Εδώ απαιτείται η δημιουργία ενός αρχείου `openssl.x509.client.conf` με περιεχόμενα:
```Shell
# These extensions are added when 'ca' signs a request.
keyUsage = digitalSignature
extendedKeyUsage=clientAuth
```

Για να δημιουργηθεί το πιστοποιητικό του κάθε πελάτη εκτελούμε για κάθε πελάτη τις εντολές:
```Shell
openssl genrsa -out client.key 4096
openssl req -new -utf8 -key client.key -out client.csr
openssl x509 -req -days 36500 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out client.crt -extfile openssl.x509.client.conf
rm client.csr
```
και προκύπτει το δημόσιο `client.crt` και το ιδιωτικό `client.key` κλειδί.

**Προσοχή!** Το OpenSSL δεν απαιτεί να συμπληρωθεί κάποιο Common Name (CA) στα στοιχεία του πιστοποιητικού. Το OpenVPN όμως το απαιτεί. Οπότε συμπληρώστε το υποχρεωτικά!


Δημιουργώντας το tls-auth κλειδί
--------------------------------
Για να δημιουργηθεί το κλειδί tls-auth το οποίο παρέχει ένα μεγαλύτερο επίπεδο ασφάλειας εκτελούμε τις εντολές:
```Shell
openvpn --genkey --secret ta.key
```
και προκύπτει το ιδιωτικό `ta.key` κλειδί.


Δημιουργώντας το αρχείο Diffie-Hellman
--------------------------------------
Για να δημιουργηθεί το αρχείο Diffie-Hellman 4096 bits εκτελούμε τις εντολές:
```Shell
openssl dhparam -out dh4096.pem 4096
```
και προκύπτει -μετά από πάρα πολύ ώρα (ώρες ή μέρες)- το αρχείο `dh4096.pem`.


Στήνοντας τον εξυπηρετητή TAP
-----------------------------
Τροποποιούμε το αρχείο ρυθμίσεων δικτύου στη διεύθυνση `/etc/network/interfaces` όπως παρακάτω:
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
Απενεργοποιούμε οτιδήποτε έχει σχέση με το `eth0` και ενεργοποιούμε τη γέφυρα δικτύου `br0` με τα ίδια χαρακτηριστικά.
Εκεί που λέει `bridge_ports` δεν μπορούμε να βάλουμε και το `tap0` (η διασύνδεση δικτύου που δημιουργεί το OpenVPN) γιατί δεν έχει εκτελεστεί ακόμα ο OpenVPN εξυπηρετητής για να το δημιουργήσει. Η ρύθμιση δικτύου είναι προαπαιτούμενο του OpenVPN εξυπηρετητή.

Μεταφέρουμε τα αρχεία `ca.crt`, `dh4096.pem`, `server.crt`, `server.key`, `ta.key`, στο φάκελο `/etc/openvpn`, με κυριότητα `root:root` και εξουσιοδότηση `400`.

Δημιουργούμε ένα **εκτελέσιμο** αρχείο `/etc/openvpn/openvpn_up` με περιεχόμενα:
```Shell
#!/bin/bash
brctl addif br0 $1
ifconfig $1 up
```
Το αρχείο αυτό, μόλις ολοκληρωθεί η δημιουργία της διασύνδεσης δικτύου `tap0` (`$1`) από το OpenVPN, τότε συνδέει τη διασύνδεση δικτύου με τη γέφυρα δικτύου `br0`, την οποία δεν έχουμε δημιουργήσει ακόμα. Ακολούθως, ενεργοποιεί τη διασύνδεση δικτύου `tap0`.

Ρυθμίζουμε ή δημιουργούμε το αρχείο `/etc/openvpn/server_tap.conf` όπως παρακάτω (τα σχόλια αφαιρέθηκαν):
```Shell
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
;down openvpn_down
```

**Παρατηρήσεις:**
- Σε όλους τους VPN TAP πελάτες γνωστοποιούμε το VPN TUN δίκτυο `10.8.0.0`.

Κάνουμε επανεκκίνηση.


Στήνοντας τον εξυπηρετητή TUN
-----------------------------
Χρησιμοποιούμε τα ίδια αρχεία `ca.crt`, `dh4096.pem`, `server.crt`, `server.key`, `ta.key` με τον εξυπηρετητή TAP οπότε δεν αλλάζουμε κάτι.

Ρυθμίζουμε ή δημιουργούμε το αρχείο `/etc/openvpn/server_tun.conf` όπως παρακάτω (τα σχόλια αφαιρέθηκαν):
```Shell
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

**Παρατηρήσεις:**
- Χρησιμοποιούμε την επόμενη πόρτα για το TUN VPN `1195`.
- Δεν τρέχουμε το εκτελέσιμο αρχείο που θα γεφύρωνε στο `br0` το `tun0` με τα `tap0` και `eth0` γιατί πολύ απλά το `tun0` είναι layer 3 και δε γεφυρώνεται.
- Σε όλους τους VPN TUN πελάτες γνωστοποιούμε το VPN TAP δίκτυο `10.0.0.0`.

Όπως αναφέρθηκε, οι συσκευές του ίδιου φυσικού δικτύου με τον εξυπηρετητή, δεν εκτελούν VPN TAP πελάτη οπότε δεν γνωρίζουν καθόλου για το δίκτυο TUN (`10.8.0.0/24`).

Προκειμένου να φτάσουν στον προορισμό τους τα πακέτα δεδομένων που έρχονται στον εξυπηρετητή από τις παραπάνω συσκευές, με προορισμό κάποια συσκευή του VPN TUN δικτύου (`10.8.0.0/24`) και αντίστροφα, θα πρέπει να είναι ενεργό το IP forwarding.
Για να ελέγξουμε αν είναι ενεργό εκτελούμε την εντολή:
```Shell
cat /proc/sys/net/ipv4/ip_forward
```
Αν λάβουμε 1 (ναι) καλώς. Αν λάβουμε 0 (όχι) πρέπει να το ενεργοποιήσουμε.

Επεξεργαζόμαστε το αρχείο /etc/sysctl.conf και αφαιρούμε το σχολιασμό από τη γραμμή:
```INI
net.ipv4.ip_forward=1
```
ενώ για να ενημερωθεί το σύστημα για την αλλαγή, εκτελούμε την εντολή:
```Shell
sysctl -p
```
Κάνουμε επανεκκίνηση.


Στήνοντας τον πελάτη TAP
------------------------
Φέρνουμε τα αρχεία `ca.crt`, `dh4096.pem`, `client.crt`, `client.key`, `ta.key` στο φάκελο `/etc/openvpn`, με κυριότητα `root:root` και εξουσιοδότηση `400`.

Αν έχουμε Windows στον πελάτη, κινούμαστε αντίστοιχα.

Ρυθμίζουμε ή δημιουργούμε το αρχείο `/etc/openvpn/client.conf` όπως παρακάτω (τα σχόλια αφαιρέθηκαν):
```INI
remote myserver.myftp.org 1194
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

Εναλλακτικά μπορούμε να εισάγουμε όλα τα πιστοποιητικά στο αρχείο ρυθμίσεων το οποίο θα γίνει κάπως έτσι:
```INI
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

Εκτελούμε τον πελάτη OpenVPN.


Στήνοντας τον πελάτη TUN
------------------------
Φέρνουμε τα αρχεία `ca.crt`, `dh4096.pem`, `client.crt`, `client.key`, `ta.key` στο φάκελο `/etc/openvpn`, με κυριότητα `root:root` και εξουσιοδότηση `400`.
Αν έχουμε Windows στον πελάτη, κινούμαστε αντίστοιχα.

Ρυθμίζουμε ή δημιουργούμε το αρχείο `/etc/openvpn/client.conf` όπως παρακάτω (τα σχόλια αφαιρέθηκαν):
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

Εναλλακτικά μπορούμε να εισάγουμε όλα τα πιστοποιητικά στο αρχείο ρυθμίσεων με την ίδια λογική που το κάναμε και στο TAP.

Εκτελούμε τον πελάτη OpenVPN.


Συνδέοντας συσκευές του ίδιου φυσικού δικτύου με τον εξυπηρετητή, που δεν εκτελούν TAP ή TUN πελάτη, με το VPN TUN
------------------------------------------------------------------------------------------------------------------
Αν η συσκευή εκτελεί Windows, εκτελούμε την εντολή:
```Shell
route -p add 10.8.0.0 mask 255.255.255.0 10.0.0.2
```
Αν η συσκευή εκτελεί Linux, κάτι αντίστοιχο.
