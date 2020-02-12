---
title:  "VPN con OpenVPN y certificados x509"
excerpt: "Creación de un escenario cliente-servidor con OpenVPN"
date:   2020-01-27 12:19:00
categories: [VPN,SSL]
---

## Introducción

**OpenVPN** es un paquete que ofrece un servicio de Redes Privadas Virtuales (VPN). Estas redes sirven para interconectar dos redes distintas de manera que puedan tener conectividad como si estuviesen dentro de la misma red local.
Además de esta funcionalidad, aporta un sistema de cifrado de la conexión y una gran flexibilidad a la hora de configurar el tipo de conexión que deseemos (tcp, udp, mayor o menor cifrado, etc).
En el siguiente supuesto práctico vamos a interconectar dos máquinas que están en redes separadas, y para ello vamos a crear una _VPN_ con el rango **10.99.99.0/32**. Dividiremos la configuración en dos partes; la configuración del lado del servidor, que también se comportará como autoridad certificadora, y la configuración del lado del cliente.

### Configuración del servidor y CA

* **Atención**: Esta configuración está hecha sobre una máquina _Vagrant_ con el sistema _Debian Buster_ y con el script **easyrsa3**. Si queréis replicar el ejercicio os dejo por aquí el fichero vagrant del [servidor](/docs/EscenarioVPN/Vagrantfile) y el del [cliente](/docs/EscenarioVPNclient/Vagrantfile).

Para hacer esto vamos a hacer uso del repositorio que nos da _OpenVPN_ y que contiene los _scripts_ necesarios para generar todas las claves, tanto la clave autofirmada, como las claves del cliente y del servidor. Por norma general necesitaremos las siguientes credenciales _como mínimo_.
* Autoridad certificadora (CA)
    * Certificado - **.crt**
    * Clave privada - **.key**
* Servidor
    * Certificado - **.crt**
    * Clave privada - **.key**
* Clave [Diffie Hellman](https://es.wikipedia.org/wiki/Diffie-Hellman) - **.pem**

#### Preparación del entorno

Vamos a necesitar el paquete **git** por lo que lo instalamos con **apt** y acto seguido clonamos el [repositorio](https://github.com/OpenVPN/easy-rsa). Por supuesto también necesitaremos instalar el paquete **openvpn**.

{% highlight bash %}
apt install git
apt install openvpn
git clone https://github.com/OpenVPN/easy-rsa
{% endhighlight %}

Entramos en el directorio generado y vamos a empezar por declarar las variables que utilizaremos para generar los certificados. Para ello copiamos el fichero **./easyrsa3/vars.example** en **./easyrsa3/vars** y modificamos las siguientes lineas.

{% highlight bash %}
# Organizational fields (used with 'org' mode and ignored in 'cn_only' mode.)
# These are the default values for fields which will be placed in the
# certificate.  Don't leave any of these fields blank, although interactively
# you may omit any specific field by typing the "." symbol (not valid for
# email.)

set_var EASYRSA_REQ_COUNTRY     "ES"
set_var EASYRSA_REQ_PROVINCE    "Sevilla"
set_var EASYRSA_REQ_CITY        "Dos Hermanas"
set_var EASYRSA_REQ_ORG         "iesgn.org"
set_var EASYRSA_REQ_EMAIL       "luis@iesgn.org"
set_var EASYRSA_REQ_OU          "iesgn.org"
{% endhighlight %}

_Estas variables tendrás que cambiarlas según tus necesidades_
A partir de ahora vamos a utilizar el _script_ **./easyrsa3/easyrsa**. Primero vamos a generar el directorio **./pki/** donde se almacenarán todas las claves y certificados.

{% highlight bash %}
root@servidor:/home/vagrant/easy-rsa/easyrsa3# chmod 700 ./easyrsa
root@servidor:/home/vagrant/easy-rsa/easyrsa3# ./easyrsa init-pki

Note: using Easy-RSA configuration from: /home/vagrant/easy-rsa/easyrsa3/vars

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /home/vagrant/easy-rsa/easyrsa3/pki
{% endhighlight %}

#### Credenciales CA

Una vez hecho esto procedemos a generar todas las claves. Empezaremos creando las credenciales de la _CA_ y lo haremos utilizando el parámetro **build-ca**.

{% highlight bash %}
root@servidor:/home/vagrant/easy-rsa/easyrsa3# ./easyrsa build-ca

Note: using Easy-RSA configuration from: /home/vagrant/easy-rsa/easyrsa3/vars
Using SSL: openssl OpenSSL 1.1.1d  10 Sep 2019

Enter New CA Key Passphrase: 
Re-Enter New CA Key Passphrase: 
Generating RSA private key, 2048 bit long modulus (2 primes)
.............+++++
...............................................................+++++
e is 65537 (0x010001)
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:iesgn.org

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/home/vagrant/easy-rsa/easyrsa3/pki/ca.crt
{% endhighlight %}

Durante este proceso nos pedirá una clave, que será la frase de paso de la clave privada de la CA. Clave que nos pedirá en el siguiente paso para generar las credenciales del servidor.

#### Credenciales Servidor

El próximo parámetro que utilizaremos con el _script_ será **build-server-full** seguido del nombre que queramos que tengan los ficheros. Primero nos pedirá una frase de paso para proteger la clave privada del servidor, y luego nos pedirá la frase de paso de la _CA_ para que pueda firmar dicha clave.

{% highlight bash %}
root@servidor:/home/vagrant/easy-rsa/easyrsa3# ./easyrsa build-server-full servidor

Note: using Easy-RSA configuration from: /home/vagrant/easy-rsa/easyrsa3/vars
Using SSL: openssl OpenSSL 1.1.1d  10 Sep 2019
Generating a RSA private key
...........................................................................................................+++++
..................+++++
writing new private key to '/home/vagrant/easy-rsa/easyrsa3/pki/easy-rsa-3372.AelZRV/tmp.J3IpE5'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
Using configuration from /home/vagrant/easy-rsa/easyrsa3/pki/easy-rsa-3372.AelZRV/tmp.3XTNjT
Enter pass phrase for /home/vagrant/easy-rsa/easyrsa3/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'servidor'
Certificate is to be certified until May  2 12:25:09 2022 GMT (825 days)

Write out database with 1 new entries
Data Base Updated
{% endhighlight %}

#### Credenciales Cliente

En mi caso, voy a llamar al cliente "Fernando", y el parámetro a utilizar es muy similar al que hemos usado con el servidor. Generaremos las claves ejecutando `easyrsa build-client-full Fernando` y de la misma forma nos pedirá las dos claves, tal y como pasó antes.

{% highlight bash %}
root@servidor:/home/vagrant/easy-rsa/easyrsa3# ./easyrsa build-client-full Fernando

Note: using Easy-RSA configuration from: /home/vagrant/easy-rsa/easyrsa3/vars
Using SSL: openssl OpenSSL 1.1.1d  10 Sep 2019
Generating a RSA private key
.+++++
........+++++
writing new private key to '/home/vagrant/easy-rsa/easyrsa3/pki/easy-rsa-3486.GLtIgl/tmp.Cs3XMF'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
Using configuration from /home/vagrant/easy-rsa/easyrsa3/pki/easy-rsa-3486.GLtIgl/tmp.bqpjRz
Enter pass phrase for /home/vagrant/easy-rsa/easyrsa3/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'Fernando'
Certificate is to be certified until May  2 12:30:04 2022 GMT (825 days)

Write out database with 1 new entries
Data Base Updated
{% endhighlight %}

#### Clave Diffie Hellman

Por último solo nos queda generar esta clave, para la que **no** se nos pedirá ninguna frase de paso, ya que no requiere de la confianza de la _CA_ y que consiste en un algoritmo criptográfico, cuyo fin es muy similar a los usados en otros protocolos como _HTTPS_, es decir, cifrar de forma asimétrica la conexión. Dicho esto procedemos a generar la clave utilizando el parámetro **gen-dh**.
Aunque parezca que tarda demasiado, no os preocupéis, _Diffie-Hellman_ es un algoritmo de encriptación _duro_, y como nos indica la propia salida del comando _OpenSSL_, tarda un rato.

{% highlight bash %}
root@servidor:/home/vagrant/easy-rsa/easyrsa3# ./easyrsa gen-dh

Note: using Easy-RSA configuration from: /home/vagrant/easy-rsa/easyrsa3/vars
Using SSL: openssl OpenSSL 1.1.1d  10 Sep 2019
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
........+.................+................................................+...............................+............................................................................................................+.................+...............................................................................+.....................................+...............................................................

.
.
.

+..............................................................................................................+...........+.........................................................+......................................................+.................++*++*++*++*

DH parameters of size 2048 created at /home/vagrant/easy-rsa/easyrsa3/pki/dh.pem

{% endhighlight %}

#### Configurando fichero OpenVPN

Antes de configurar el fichero en si mismo, vamos a organizar un poco los ficheros con las credenciales. Básicamente vamos a mover los ficheros con las claves pertinentes (ubicados, en mi caso, anteriormente en **/home/vagrant/easy-rsa/easyrsa3/pki/**) al directorio de configuración de _openvpn_, es decir a **/etc/openvpn/**. El arbol de directorios que he creado es el siguiente:

{% highlight bash %}
root@servidor:/etc/openvpn# tree pki
pki
├── ca
│   ├── ca.crt
│   └── ca.key
├── dh.pem
└── server
    ├── server.crt
    └── server.key

2 directories, 5 files
{% endhighlight %}

Después generamos un fichero **.conf** que contendrá la configuración del servidor. Debería quedar algo así:

{% highlight bash %}
#Dispositivo de túnel
dev tun

#Protocolo de red
proto tcp

#Direcciones IP virtuales
server 10.99.99.0 255.255.255.0 

#subred local
push "route 192.168.100.0 255.255.255.0"

#Rol de servidor
tls-server

#Parámetros Diffie-Hellman
dh /etc/openvpn/pki/dh.pem

#Certificado de la CA
ca /etc/openvpn/pki/ca/ca.crt

#Certificado local
cert /etc/openvpn/pki/server/server.crt

#Clave privada local
key /etc/openvpn/pki/server/server.key

#Activar la compresión LZO
comp-lzo

#Detectar caídas de la conexión
keepalive 10 60

#Nivel de información
verb 3

# Contraseña del certificado del servidor
askpass contraseña.txt
{% endhighlight %}

Como indico al final en el fichero de configuración, tendremos que crear un fichero con la frase de paso de las credenciales del servidor, para que al iniciar demonio de _openvpn_ no tengamos problemas. Para terminar la configuración del servidor, reiniciamos el servicio de _openvpn_ y comprobamos que está "escuchando" por el **puerto 1194**.

{% highlight bash %}
root@servidor:/etc/openvpn#  lsof -i -P -n | grep openvpn
openvpn  3680    root    6u  IPv4  28772      0t0  TCP *:1194 (LISTEN)
{% endhighlight %}

### Configuración del cliente

Primero necesitaremos las credenciales que generamos en el lado del servidor. Puedes transferirlas con **scp** por ejemplo. En resumen el cliente necesitará:
* Credenciales "Fernando"
    * Certificado - **.crt**
    * Clave privada - **.key**
* Certificado CA - **.crt**

Tal y como hicimos en el servidor, instalamos el paquete **openvpn** con **apt**. Después nos dirigimos al directorio **/etc/openvpn** y generamos un fichero de configuración que llamaremos **cliente.conf** y que tendrá el siguiente contenido.

{% highlight bash %}
#Dispositivo de túnel
dev tun

#IP del servidor
remote 172.22.6.87 

#Direcciones IP virtuales
ifconfig 10.99.99.0 255.255.255.0
pull

#Protocolo
proto tcp-client # Protocolo tcp

#Rol de cliente
tls-client

#Certificado de la CA
ca /etc/openvpn/pki/CA/ca.crt

#Certificado cliente
cert /etc/openvpn/pki/client/fernando.crt

#Clave privada cliente
key /etc/openvpn/pki/client/fernando.key

#Activar la compresión LZO
comp-lzo

#Detectar caídas de la conexión
keepalive 10 60

#Nivel de la información
verb 3

# Contraseña del certificado del cliente
askpass contraseña.txt
{% endhighlight %}

Antes de hacer que funcione con el demonio de _OpenVPN_, vamos a probar ejecutandolo directamente con el comando, seguido del fichero de configuración, y comprobamos la conectividad con la máquina de la red local.

{% highlight bash %}
root@cliente:/etc/openvpn# openvpn --config cliente.conf &
[1] 999
root@cliente:/etc/openvpn# Tue Jan 28 13:33:32 2020 WARNING: file 'contraseña.txt' is group or others accessible
Tue Jan 28 13:33:32 2020 OpenVPN 2.4.7 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on Feb 20 2019
Tue Jan 28 13:33:32 2020 library versions: OpenSSL 1.1.1c  28 May 2019, LZO 2.10
Tue Jan 28 13:33:32 2020 WARNING: using --pull/--client and --ifconfig together is probably not what you want
Tue Jan 28 13:33:32 2020 WARNING: No server certificate verification method has been enabled.  See http://openvpn.net/howto.html#mitm for more info.
Tue Jan 28 13:33:32 2020 WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
Tue Jan 28 13:33:32 2020 TCP/UDP: Preserving recently used remote address: [AF_INET]172.22.6.87:1194
Tue Jan 28 13:33:32 2020 Socket Buffers: R=[87380->87380] S=[16384->16384]
Tue Jan 28 13:33:32 2020 Attempting to establish TCP connection with [AF_INET]172.22.6.87:1194 [nonblock]
Tue Jan 28 13:33:33 2020 TCP connection established with [AF_INET]172.22.6.87:1194
Tue Jan 28 13:33:33 2020 TCP_CLIENT link local: (not bound)
Tue Jan 28 13:33:33 2020 TCP_CLIENT link remote: [AF_INET]172.22.6.87:1194
Tue Jan 28 13:33:33 2020 TLS: Initial packet from [AF_INET]172.22.6.87:1194, sid=75408fc6 10698460
Tue Jan 28 13:33:34 2020 VERIFY OK: depth=1, CN=iesgn.org
Tue Jan 28 13:33:34 2020 VERIFY OK: depth=0, CN=server
Tue Jan 28 13:33:34 2020 Control Channel: TLSv1.3, cipher TLSv1.3 TLS_AES_256_GCM_SHA384, 2048 bit RSA
Tue Jan 28 13:33:34 2020 [server] Peer Connection Initiated with [AF_INET]172.22.6.87:1194
Tue Jan 28 13:33:35 2020 SENT CONTROL [server]: 'PUSH_REQUEST' (status=1)
Tue Jan 28 13:33:35 2020 PUSH: Received control message: 'PUSH_REPLY,route 192.168.100.0 255.255.255.0,route 10.99.99.1,topology net30,ping 10,ping-restart 60,ifconfig 10.99.99.6 10.99.99.5,peer-id 0,cipher AES-256-GCM'
Tue Jan 28 13:33:35 2020 OPTIONS IMPORT: timers and/or timeouts modified
Tue Jan 28 13:33:35 2020 OPTIONS IMPORT: --ifconfig/up options modified
Tue Jan 28 13:33:35 2020 OPTIONS IMPORT: route options modified
Tue Jan 28 13:33:35 2020 OPTIONS IMPORT: peer-id set
Tue Jan 28 13:33:35 2020 OPTIONS IMPORT: adjusting link_mtu to 1627
Tue Jan 28 13:33:35 2020 OPTIONS IMPORT: data channel crypto options modified
Tue Jan 28 13:33:35 2020 Data Channel: using negotiated cipher 'AES-256-GCM'
Tue Jan 28 13:33:35 2020 Outgoing Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Tue Jan 28 13:33:35 2020 Incoming Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Tue Jan 28 13:33:35 2020 ROUTE_GATEWAY 172.22.0.1/255.255.0.0 IFACE=eth1 HWADDR=08:00:27:9b:bf:06
Tue Jan 28 13:33:35 2020 TUN/TAP device tun0 opened
Tue Jan 28 13:33:35 2020 TUN/TAP TX queue length set to 100
Tue Jan 28 13:33:35 2020 /sbin/ip link set dev tun0 up mtu 1500
Tue Jan 28 13:33:35 2020 /sbin/ip addr add dev tun0 local 10.99.99.6 peer 10.99.99.5
Tue Jan 28 13:33:35 2020 /sbin/ip route add 192.168.100.0/24 via 10.99.99.5
Tue Jan 28 13:33:35 2020 /sbin/ip route add 10.99.99.1/32 via 10.99.99.5
Tue Jan 28 13:33:35 2020 Initialization Sequence Completed

root@cliente:/etc/openvpn# ping -c 1 192.168.100.20
PING 192.168.100.20 (192.168.100.20) 56(84) bytes of data.
64 bytes from 192.168.100.20: icmp_seq=1 ttl=63 time=2.64 ms

--- 192.168.100.20 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 2.643/2.643/2.643/0.000 ms
{% endhighlight %}

Si queremos que funcione como _demonio_ del sistema, nos dirigimos al fichero **/etc/default/openvpn** y modificamos la siguiente linea.

{% highlight bash %}
AUTOSTART="cliente"
{% endhighlight %}

Deberás especificar la o las configuraciones vpn que quieras iniciar con el demonio. El nombre de estas es igual que el de los ficheros de configuración sin la extensión **.conf**. Como en mi caso he creado el fichero de configuración **cliente.conf** la _VPN_ se llama **cliente**. También podríamos levantarlas todas a la vez especificando **ALL**.

## Configuración VPN sitio a sitio

A diferencia del escenario anterior, una **VPN sitio a sitio** se basa en un concepto algo distinto. Mientras que en el primer caso, hemos conectado _una máquina_ a _una red remota_, en este caso vamos a conectar **dos redes remotas**, por lo que técnicamente, estaremos fusionando las dos redes a _nivel de aplicación_.
En este nuevo escenario, tendremos dos máquinas en redes locales diferentes; por un lado la **192.168.100.0/24** y por otro lado la **192.168.200.0/24**. A su vez, cada una de estas máquinas estará compartiendo la red con un servidor que serán los que gestionen la conexión de la **VPN**, que tendrán una red común, la **172.22.0.0/16**. Por lo tanto el exquema de la red es el siguiente:

![](/images/vpn-site-to-site/)

Si ya os habéis peleado con la configuración de la **VPN cliente-servidor**, esta os resultará bastante familiar. Los ficheros de configuración son muy similares, y la única diferencia reside en el encaminamiento o _enrutamiento_ que definimos. Aunque en una primera instancia podamos pensar que en este escenario va a haber dos servidores. La conexión funcionara prácticamente igual que en el caso anterior, donde habrá un _servidor_ y un _cliente_, salvo que este último funcionará también como el _servidor_ de su _red local_. Vamos a ver el fichero de configuración del servidor (en el esquema sería **R2**):

{% highlight bash %}
# Dispositivo de túnel
dev tun
# Encaminamiento
ifconfig 10.99.99.1 10.99.99.2
route 192.168.200.0 255.255.255.0
# Rol de cliente
tls-server

# Certificado Diffie-Hellman
dh /etc/openvpn/pki/dh.pem
# Certificado de la CA
ca /etc/openvpn/pki/ca.crt
# Certificado local
cert /etc/openvpn/pki/issued/servidor.crt
# Clave privada local
key /etc/openvpn/pki/private/servidor.key

# Activar la compresión LZO
comp-lzo

# Detectar caídas de la conexión
keepalive 10 60

# Nivel de información
verb 3

askpass contra.txt
{% endhighlight %}

Como podemos observar, si comparamos con el fichero de configuración de la **VPN cliente-servidor**, lo único que cambia son estas lineas

{% highlight bash %}
# CLIENTE-SERVIDOR
#Direcciones IP virtuales
server 10.99.99.0 255.255.255.0 

#subred local
push "route 192.168.100.0 255.255.255.0"

# SITE-TO-SITE
# Encaminamiento
ifconfig 10.99.99.1 10.99.99.2
route 192.168.200.0 255.255.255.0
{% endhighlight %}

Tenemos dos campos distintos.
* **ifconfig**: definirá la _ip virtual origen_ del servidor en primer lugar, y en segundo lugar la _ip virtual del "cliente" remoto_.
* **route**: define la _red física remota_ a la que queremos conectarnos.

En el lado del cliente nos encontramos con la siguiente configuración:

{% highlight bash %}
# Dispositivo de túnel
dev tun
# Encaminamiento
ifconfig 10.99.99.2 10.99.99.1
# Direcciones IP virtuales
remote 172.22.6.24 
# Subred remota
route 192.168.100.0 255.255.255.0
# Rol de cliente
tls-client

# Certificado de la CA
ca /etc/openvpn/pki/ca-client/ca.crt

#Certificado local
cert /etc/openvpn/pki/server-client/Luis.crt

#Clave privada local
key /etc/openvpn/pki/server-client/Luis.key

#Activar la compresión LZO
comp-lzo

#Detectar caídas de la conexión
keepalive 10 60

#Nivel de información
verb 3

askpass password.txt
{% endhighlight %}

En este caso podemos detectar tres campos clave que han cambiado, y son:
* **ifconfig**: Tal y como hemos explicado antes, define la _ip virtual origen_ del servidor en primer lugar, y la _ip virtual del "cliente" remoto_.
* **remote**: Aquí tenemos que especificar la dirección IP, del servidor **VPN**. Si ambos servidores no contasen con una red local común y simplemente estuviesen conectados a internet, aquí tendríamos que colocar la _IP pública_ o el respectivo _DNS_ de la máquina.
* **route**: De la misma forma que en el servidor, indicamos la _red física remota_ a la que nos conectaremos.

Una vez definidos, modificamos el fichero **/etc/default/openvpn** si hemos creado nuevos ficheros **.conf** para definir esta nueva conexión, y posteriormente reiniciamos los servicios. Primero el servicio del servidor, y luego el del cliente.
Debemos recordar activar el **bit de forward** en ambas máquinas, para que puedan "pasar" el tráfico sus respectivas _redes locales_. Una vez hecho esto probamos que funciona. Primero probamos un **ping** desde el _PC1_ al _PC2_.

{% highlight bash %}
vagrant@cliente:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 85522sec preferred_lft 85522sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:ae:6e:e7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.200.40/24 brd 192.168.200.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feae:6ee7/64 scope link 
       valid_lft forever preferred_lft forever
vagrant@cliente:~$ ping -c 3 192.168.100.50
PING 192.168.100.50 (192.168.100.50) 56(84) bytes of data.
64 bytes from 192.168.100.50: icmp_seq=1 ttl=62 time=3.49 ms
64 bytes from 192.168.100.50: icmp_seq=2 ttl=62 time=3.15 ms
64 bytes from 192.168.100.50: icmp_seq=3 ttl=62 time=3.83 ms

--- 192.168.100.50 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 3.150/3.487/3.826/0.280 ms
{% endhighlight %}

Luego probamos desde el _PC2_ al _PC1_.

{% highlight bash %}
vagrant@openvpnprivada:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 85777sec preferred_lft 85777sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:48:99:9a brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.50/24 brd 192.168.100.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe48:999a/64 scope link 
       valid_lft forever preferred_lft forever
vagrant@openvpnprivada:~$ ping -c 3 192.168.200.40
PING 192.168.200.40 (192.168.200.40) 56(84) bytes of data.
64 bytes from 192.168.200.40: icmp_seq=1 ttl=62 time=3.77 ms
64 bytes from 192.168.200.40: icmp_seq=2 ttl=62 time=2.94 ms
64 bytes from 192.168.200.40: icmp_seq=3 ttl=62 time=3.47 ms

--- 192.168.200.40 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 4ms
rtt min/avg/max/mdev = 2.944/3.395/3.773/0.345 ms
{% endhighlight %}

* **Referencias**: [Documentación oficial OpenVPN](https://openvpn.net/community-resources/how-to/)


