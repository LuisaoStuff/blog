---
title:  "Práctica: Cortafuegos perimetral con DMZ"
excerpt: "Creación de un cortafuegos perimetral con DMZ en iptables e introducción a nftables"
date:   2019-12-05 16:30:00
categories: [Seguridad]
tags: [iptables, nftables]
---

## Introducción

En esta entrada vamos a realizar una práctica que consistirá en la creación de un cortafuegos perimetral contando con una zona DMZ y una zona segura. Dicho cortafuegos lo implementaremos con **iptables** y más tarde lo optimizaremos haciendo uso de _cadenas_ para segmentar las reglas de cortafuegos según la dirección y sentido del tráfico que estamos filtrando.

#### Política por defecto DROP para las cadenas INPUT, FORWARD y OUTPUT.

{% highlight bash %}
#Establecemos política ACCEPT para no perder la conexión al limpiar las tablas
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT

# Limpiamos las tablas
iptables -F
iptables -t nat -F
iptables -Z
iptables -t nat -Z

#Habilitamos ssh para administrar, vamos a utilizar el módulo para filtrar por MAC, utilizando la MAC de croqueta para que solo podamos administrar el router desde allí.

iptables -A INPUT -p tcp --dport 22 -m mac --mac-source fa:16:3e:8f:8e:6e -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT

# Establecemos la política
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

#Bit de forward
echo 1 > /proc/sys/net/ipv4/ip_forward
{% endhighlight %}
    
#### La máquina router-fw tiene un servidor ssh escuchando por el puerto 22, pero al acceder desde el exterior habrá que conectar al puerto 2222.

* Redirigir peticiones del puerto 2222 al puerto 22

{% highlight bash %}
iptables -t nat -A PREROUTING -i eth0 -p tcp -m tcp --dport 2222 -j REDIRECT --to-ports 22
{% endhighlight %}

* Bloquear el acceso directo desde el puerto 22

{% highlight bash %}
iptables -t nat -A PREROUTING -i eth0 -p tcp -m tcp --dport 22 -j DNAT --to-destination 127.0.0.1:22
{% endhighlight %}

Si probamos a conectarnos desde el puerto 22, no podemos, tendremos que hacerlo usando el puerto 2222:

{% highlight bash %}
debian@croqueta:~$ ssh debian@10.0.0.6
^C
debian@croqueta:~$ ssh -p 2222 debian@10.0.0.6
Linux router-fw 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Dec 10 11:02:23 2019 from 10.0.0.3
debian@router-fw:~$ 
{% endhighlight %}

#### Desde la LAN y la DMZ se debe permitir la conexión ssh por el puerto 22 al la máquina router-fw.

* Permitir desde la LAN:

{% highlight bash %}
iptables -A INPUT -s 192.168.100.0/24 -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -d 192.168.100.0/24 -p tcp --sport 22 -j ACCEPT
{% endhighlight %}

<a><img src="/images/cortafuegos-dmz/ssh-desde-LAN.png"/></a>

* Permitir desde la DMZ:

{% highlight bash %}
iptables -A INPUT -s 192.168.200.0/24 -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -d 192.168.200.0/24 -p tcp --sport 22 -j ACCEPT
{% endhighlight %}

<a><img src="/images/cortafuegos-dmz/ssh-desde-DMZ.png"/></a>

#### La máquina router-fw debe tener permitido el tráfico para la interfaz loopback.

{% highlight bash %}
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
{% endhighlight %}

#### A la máquina router-fw se le puede hacer ping desde la DMZ, pero desde la LAN se le debe rechazar la conexión (REJECT).

* Ping desde la DMZ

{% highlight bash %}
iptables -A INPUT -i eth2 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A OUTPUT -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
{% endhighlight %}

<a><img src="/images/cortafuegos-dmz/ping-dmz.png"/></a>

* Ping desde la LAN

{% highlight bash %}
iptables -A INPUT -i eth1 -s 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-request -j REJECT --reject-with icmp-port-unreachable
iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m state --state RELATED -j ACCEPT
{% endhighlight %}

<a><img src="/images/cortafuegos-dmz/ping-lan.png"/></a>

#### La máquina router-fw puede hacer ping a la LAN, la DMZ y al exterior.

* Ping a la LAN

{% highlight bash %}
iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i eth1 -s 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
{% endhighlight %}

*Prueba*

{% highlight bash %}
root@router-fw:~# ping -c 3 192.168.100.10
PING 192.168.100.10 (192.168.100.10) 56(84) bytes of data.
64 bytes from 192.168.100.10: icmp_seq=1 ttl=64 time=0.806 ms
64 bytes from 192.168.100.10: icmp_seq=2 ttl=64 time=0.659 ms
64 bytes from 192.168.100.10: icmp_seq=3 ttl=64 time=0.776 ms

--- 192.168.100.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 9ms
rtt min/avg/max/mdev = 0.659/0.747/0.806/0.063 ms
{% endhighlight %}

* Ping a la DMZ

{% highlight bash %}
iptables -A OUTPUT -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i eth2 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
{% endhighlight %}

*Prueba*

{% highlight bash %}
root@router-fw:~# ping -c 3 192.168.200.10
PING 192.168.200.10 (192.168.200.10) 56(84) bytes of data.
64 bytes from 192.168.200.10: icmp_seq=1 ttl=64 time=1.82 ms
64 bytes from 192.168.200.10: icmp_seq=2 ttl=64 time=1.09 ms
64 bytes from 192.168.200.10: icmp_seq=3 ttl=64 time=1.09 ms

--- 192.168.200.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 1.092/1.335/1.821/0.344 ms
{% endhighlight %}


* Ping al exterior

{% highlight bash %}
iptables -A OUTPUT -o eth0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i eth0 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
{% endhighlight %}

*Prueba*

{% highlight bash %}
root@router-fw:~# ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=51 time=43.1 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=51 time=43.2 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=51 time=49.7 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 43.107/45.337/49.683/3.078 ms
{% endhighlight %}

#### Desde la máquina DMZ se puede hacer ping y conexión ssh a la máquina LAN.

Añadimos las reglas *FORWARD* con su correspondiente seguimiento de la conexión.

* SSH desde DMZ a la LAN

{% highlight bash %}
iptables -A FORWARD -i eth2 -o eth1 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
{% endhighlight %}

* Prueba ssh desde la *DMZ* a la *LAN*

<a><img src="/images/cortafuegos-dmz/ssh-from-dmz-to-lan.png"/></a>

* Ping desde DMZ a la LAN

{% highlight bash %}
iptables -A FORWARD -i eth2 -o eth1 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
{% endhighlight %}

* Prueba ping desde la *DMZ* a la *LAN*

<a><img src="/images/cortafuegos-dmz/ping-from-dmz-to-lan.png"/></a>

#### Desde la máquina LAN no se puede hacer ping, pero si se puede conectar por ssh a la máquina DMZ.

{% highlight bash %}
iptables -A FORWARD -i eth1 -o eth2 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth1 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
{% endhighlight %}

*Prueba ssh*

<a><img src="/images/cortafuegos-dmz/ssh-from-LAN-to-DMZ.png"/></a>

#### Configura la máquina router-fw para que las máquinas LAN y DMZ puedan acceder al exterior.

{% highlight bash %}
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.200.0/24 -o eth0 -j MASQUERADE
{% endhighlight %}

#### La máquina LAN se le permite hacer ping al exterior.

{% highlight bash %}
iptables -A FORWARD -i eth1 -o eth0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
{% endhighlight %}

<a><img src="/images/cortafuegos-dmz/ping-from-LAN-to-EXT.png"/></a>

#### La máquina LAN puede navegar.

Para conseguir esto, tenemos que permitirle resolución *dns* y los protocolos *http* y *https* (puertos 53, 80 y 443). De este modo crearemos una regla SNAT para enmascarar todo este tráfico y las reglas *forward* pertinentes.

{% highlight bash %}
# HTTP/S
iptables -A FORWARD -i eth1 -o eth0 -p tcp -m multiport --dports 80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -p tcp -m multiport --sports 80,443 -m state --state ESTABLISHED -j ACCEPT
# DNS
iptables -A FORWARD -i eth1 -o eth0 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
{% endhighlight %}

Para probar esto, actualizamos el sistema:

<a><img src="/images/cortafuegos-dmz/trying-update-from-LAN.png"/></a>

#### La máquina DMZ puede navegar. Instala un servidor web, un servidor ftp y un servidor de correos.

{% highlight bash %}
# HTTP/S
iptables -A FORWARD -i eth2 -o eth0 -p tcp -m multiport --dports 80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -p tcp -m multiport --sports 80,443 -m state --state ESTABLISHED -j ACCEPT
# DNS
iptables -A FORWARD -i eth2 -o eth0 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
{% endhighlight %}

Acto seguido instalamos los siguientes paquetes:

{% highlight bash %}
apt update
apt install apache2 postfix proftpd
{% endhighlight %}

* Servidor web: *apache2*

* Servidor de correos: *postfix*

* Servidor ftp: *proftpd*

En este servicio simplemente vamos a configurar el acceso anónimo para evitar tener que crear usuarios adicionales. Esto se hace modificando el fichero */etc/proftpd/* y descomentando estas lineas:

{% highlight bash %}
<Anonymous /srv/doc>
   User				ftp
   Group			nogroup
   # We want clients to be able to login with "anonymous" as well as "ftp"
   UserAlias			anonymous ftp
   # Cosmetic changes, all files belongs to ftp user
   DirFakeUser	on ftp
   DirFakeGroup on ftp
 
   RequireValidShell		off
 
   # Limit the maximum number of anonymous logins
   MaxClients			10
 
   # We want 'welcome.msg' displayed at login, and '.message' displayed
   # in each newly chdired directory.
   DisplayLogin			welcome.msg
   DisplayChdir		.message
 
   # Limit WRITE everywhere in the anonymous chroot
   <Directory *>
     <Limit WRITE>
       DenyAll
     </Limit>
   </Directory>
 
   # Uncomment this if you're brave.
   # <Directory incoming>
   #   # Umask 022 is a good standard umask to prevent new files and dirs
   #   # (second parm) from being group and world writable.
   #   Umask				022  022
   #            <Limit READ WRITE>
   #            DenyAll
   #            </Limit>
   #            <Limit STOR>
   #            AllowAll
   #            </Limit>
   # </Directory>

 </Anonymous>
{% endhighlight %}

Después simplemente reiniciamos el servicio con:

{% highlight bash %}
systemctl restart proftpd
{% endhighlight %}

#### Configura la máquina router-fw para que los servicios web y ftp sean accesibles desde el exterior.

Para que puedan ser accesibles, necesitaremos una regla DNAT que redirija el tráfico de los puertos 20,21,80 y 443 a la máquina de la DMZ.

{% highlight bash %}
iptables -t nat -A PREROUTING -i eth0 -p tcp -m multiport --dports 80,443 -j DNAT --to 192.168.200.10

iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 21 -j DNAT --to 192.168.200.10:21
iptables -t nat -A POSTROUTING -o eth2 -p tcp --dport 21 -d 192.168.200.10 -j SNAT --to 192.168.200.2


iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 20 -j DNAT --to 192.168.200.10:21
iptables -t nat -A POSTROUTING -o eth2 -p tcp --dport 20 -d 192.168.200.10 -j SNAT --to 192.168.200.2
{% endhighlight %}

Y las reglas FORWARD

{% highlight bash %}
iptables -A FORWARD -i eth0 -o eth2 -p tcp -m multiport --dports 80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -p tcp -m multiport --sports 80,443 -m state --state ESTABLISHED -j ACCEPT

iptables -A FORWARD -i eth0 -o eth2 -p tcp --syn --dport 21 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -p tcp --syn --dport 20 -m conntrack --ctstate NEW -j ACCEPT

iptables -A FORWARD -i eth0 -o eth2  -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0  -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
{% endhighlight %}

Prueba de accesibilidad al servidor web:

<a><img src="/images/cortafuegos-dmz/dmz-apache.png"/></a>

Prueba del servidor ftp:

{% highlight bash %}
luis@kutulu:~$ ftp 172.22.200.56
Connected to 172.22.200.56.
220 ProFTPD Server (Debian) [::ffff:192.168.200.10]
Name (172.22.200.56:luis): ftp
331 Anonymous login ok, send your complete email address as your password
Password:
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> pwd
257 "/" is the current directory
ftp> ls
500 Illegal PORT command
ftp: bind: Address already in use
ftp> cd dir1
250 CWD command successful
ftp> pwd
257 "/dir1" is the current directory
{% endhighlight %}

#### El servidor web y el servidor ftp deben ser accesible desde la LAN y desde el exterior.

Debemos aplicar unas reglas *FORWARD* similares a las anteriores, solo que teniendo como origen de las peticiones la red *LAN*

{% highlight bash %}
iptables -A FORWARD -i eth1 -o eth2 -p tcp -m multiport --dports 80,443 -d 192.168.200.10 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth1 -p tcp -m multiport --sports 80,443 -s 192.168.200.10 -m state --state ESTABLISHED -j ACCEPT

iptables -A FORWARD -i eth1 -o eth2 -p tcp --syn --dport 21 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -p tcp --syn --dport 20 -m conntrack --ctstate NEW -j ACCEPT

iptables -A FORWARD -i eth1 -o eth2  -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth1  -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
{% endhighlight %}

* Prueba acceso servidor *FTP* desde la *LAN*

<a><img src="/images/cortafuegos-dmz/ftp-from-lan-to-dmz.png"/></a>

* Prueba acceso servidor *WEB* desde la *LAN*

<a><img src="/images/cortafuegos-dmz/curl-from-lan-to-dmz.png"/></a>

#### El servidor de correos sólo debe ser accesible desde la LAN.

Para que sea accesible desde la LAN, en la instalación elegimos *internet site*. Además tendremos que especificar el el fichero de configuración */etc/postfix/main.cnf* las redes desde las que se permitirán conexiones. Esto se hace modificando la siguiente linea:

{% highlight bash %}
...
mynetworks = 127.0.0.0/8 192.168.100.0/24
...
{% endhighlight %}

Además tendremos que crear las respectivas reglas *FORWARD* que funcionen con el puerto *25*.

{% highlight bash %}
iptables -A FORWARD -i eth1 -o eth2 -p tcp --dport 25 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth1 -p tcp --sport 25 -m state --state ESTABLISHED -j ACCEPT
{% endhighlight %}

<a><img src="/images/cortafuegos-dmz/prueba-postfix-from-LAN.png"/></a>

#### En la máquina LAN instala un servidor mysql. A este servidor sólo se puede acceder desde la DMZ.

En la *LAN* instalaremos solo <code>mariadb-server</code> mientras en en la *DMZ* instalaremos <code>mariadb-client</code> para probar el acceso remoto. Una vez hecho esto, en el lado del servidor, es decir, en la LAN, accederemos a *mariadb* desde el usuario *root* con: <code>mysql -u root</code> y ejecutaremos los siguientes comandos:

{% highlight sql %}
create database pruebaFW;
create user dmz@"192.168.200.10" identified by "dios";
grant all privileges on pruebaFW.* to DMZ;
flush privileges;
{% endhighlight %}

También nos dirigiremos al fichero */etc/mysql/mariadb.conf.d/50-server.cnf* y editaremos la directiva *bind-address* para que quede así:

{% highlight bash %}
...
bind-address            = 0.0.0.0
...
{% endhighlight %}

Ahora solo nos queda crear las reglas *FORWARD*:

{% highlight bash %}
iptables -A FORWARD -i eth2 -o eth1 -p tcp --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -p tcp --sport 3306 -m state --state ESTABLISHED -j ACCEPT
{% endhighlight %}

Prueba de funcionamiento:

<a><img src="/images/cortafuegos-dmz/mysql-from-dmz-to-lan.png"/></a>

#### Debemos implementar que el cortafuego funcione después de un reinicio de la máquina.

Para hacer esto vamos a crear una unidad de *systemd* que en realidad es algo bastante sencillo de hacer. Primero crearemos un _script_ con todas las reglas de iptables, puestas en secuencia. Este sería el contenido del script:

{% highlight bash %}
#!/bin/sh

# Establecemos política ACCEPT para no perder la conexión al limpiar las tablas
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT


# Limpiamos las tablas
iptables -F
iptables -t nat -F
iptables -Z
iptables -t nat -Z


# Habilitamos ssh para administrar, vamos a utilizar el módulo para filtrar por MAC,
# utilizando la MAC de croqueta para que solo podamos administrar el router desde allí.
iptables -A INPUT -p tcp --dport 22 -m mac --mac-source fa:16:3e:8f:8e:6e -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT


# Establecemos la política
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

# Bit de forward
echo 1 > /proc/sys/net/ipv4/ip_forward


# Redirigimos el tráfico directo del puerto 2222 al 22
iptables -t nat -A PREROUTING -i eth0 -p tcp -m tcp --dport 2222 -j REDIRECT --to-ports 22


# Redirigimos el tráfico directo del puerto 22 a la loopback para inutilizarlo
iptables -t nat -A PREROUTING -i eth0 -p tcp -m tcp --dport 22 -j DNAT --to-destination 127.0.0.1:22


# Permitir SSH desde la LAN
iptables -A INPUT -s 192.168.100.0/24 -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -d 192.168.100.0/24 -p tcp --sport 22 -j ACCEPT


# Permitir  SSH desde la DMZ
iptables -A INPUT -s 192.168.200.0/24 -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -d 192.168.200.0/24 -p tcp --sport 22 -j ACCEPT


# Permitir tráfico loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT


# Permitir ping desde la DMZ
iptables -A INPUT -i eth2 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A OUTPUT -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT


# Rechazar ping desde la LAN 
iptables -A INPUT -i eth1 -s 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-request -j REJECT --reject-with icmp-port-unreachable


# Permitir ping a la LAN
iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m state --state RELATED -j ACCEPT
iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i eth1 -s 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT


# Permitir ping a la DMZ
iptables -A OUTPUT -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i eth2 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT


# Permitir ping al exterior
iptables -A OUTPUT -o eth0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i eth0 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT


# Permitir ping y ssh desde la DMZ a la LAN
iptables -A FORWARD -i eth2 -o eth1 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth1 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT


# Permitir ssh desde la LAN a la DMZ
iptables -A FORWARD -i eth1 -o eth2 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth1 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT


#Permitir acceso de DMZ y LAN al exterior
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.200.0/24 -o eth0 -j MASQUERADE


# Permitir ping desde la LAN al exterior
iptables -A FORWARD -i eth1 -o eth0 -p icmp -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -p icmp -m state --state ESTABLISHED -j ACCEPT


# Permitir navegación web desde la LAN
iptables -A FORWARD -i eth1 -o eth0 -p tcp -m multiport --dports 80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -p tcp -m multiport --sports 80,443 -m state --state ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT


# Permitir navegación web desde la DMZ
iptables -A FORWARD -i eth2 -o eth0 -p tcp -m multiport --dports 80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -p tcp -m multiport --sports 80,443 -m state --state ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT


# Permitir acceso al servidor WEB de la DMZ desde el exterior
iptables -A FORWARD -i eth0 -o eth2 -p tcp -m multiport --dports 80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -p tcp -m multiport --sports 80,443 -m state --state ESTABLISHED -j ACCEPT


# Permitir acceso al servidor FTP de la DMZ desde el exterior
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 21 -j DNAT --to 192.168.200.10:21
iptables -t nat -A POSTROUTING -o eth2 -p tcp --dport 21 -d 192.168.200.10 -j SNAT --to 192.168.200.2
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 20 -j DNAT --to 192.168.200.10:21
iptables -t nat -A POSTROUTING -o eth2 -p tcp --dport 20 -d 192.168.200.10 -j SNAT --to 192.168.200.2
iptables -A FORWARD -i eth0 -o eth2 -p tcp --syn --dport 21 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -p tcp --syn --dport 20 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -p tcp -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -p tcp -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT


# Permitir acceso al servidor WEB de la DMZ desde la LAN
iptables -A FORWARD -i eth1 -o eth2 -p tcp -m multiport --dports 80,443 -d 192.168.200.10 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth1 -p tcp -m multiport --sports 80,443 -s 192.168.200.10 -m state --state ESTABLISHED -j ACCEPT


#Permitir acceso al servidor FTP de la DMZ desde la LAN
iptables -A FORWARD -i eth1 -o eth2 -p tcp --syn --dport 21 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -p tcp --syn --dport 20 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -p tcp -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth1 -p tcp -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT


# Permitir acceso al servidor de correos de la DMZ desde la LAN
iptables -A FORWARD -i eth1 -o eth2 -p tcp --dport 25 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth1 -p tcp --sport 25 -m state --state ESTABLISHED -j ACCEPT


# Permitir acceso al servidor MYSQL de la LAN desde la DMZ
iptables -A FORWARD -i eth2 -o eth1 -p tcp --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -p tcp --sport 3306 -m state --state ESTABLISHED -j ACCEPT

nat=$(iptables -t nat -S | grep -v '^-P' | wc -l)
rules=$(iptables -S | grep -v '^-P' | wc -l)
total=$(($nat+$rules))

triedrules=$(cat /usr/bin/boot-firewall | grep '^iptables.*' | grep -v '.*-P.*\|.*-Z.*\|.*-F.*' | wc -l)

if [ $total -eq $triedrules ]; then
	echo "se han aplicado todas las reglas"
	exit 0
else
	echo "algo salió mal"
	exit 1
fi
{% endhighlight %}

Le daremos al fichero permisos de ejecución. Por seguridad le daré solo permisos de ejecución y lectura al propietario:

{% highlight bash %}
chmod 500 boot-firewall
{% endhighlight %}

Acto seguido, sitúo el _script_ en */usr/bin* y creo la unidad de systemd, que llamaré *boot-firewall.service* y la pondré en el directorio */etc/systemd/system/*:

{% highlight bash %}
[Unit]
Description=iptables rules on boot
After=networking.service
StartLimitIntervalSec=0

[Service]
Type=oneshot
RemainAfterExit=True
User=root
ExecStart=/usr/bin/boot-firewall

[Install]
WantedBy=multi-user.target
{% endhighlight %} 

En la directiva *After* indicamos después de que servicio podrá ejecutarse, y aquí le indico que se ejecute después de que se levanten las interfaces con *networking.service*. También le indico la ruta absoluta del script y qué usuario lo ejecutará.
Después simplemente tengo que iniciar el servicio:

{% highlight bash %}
systemctl start boot-firewall
{% endhighlight %}

Y habilito que se ejecute en el inicio:

{% highlight bash %}
systemctl enable boot-firewall
{% endhighlight %}

Compruebo que se ha ejecutado correctamente:

{% highlight bash %}
root@router-fw:~# systemctl status boot-firewall 
● boot-firewall.service - iptables rules on boot
   Loaded: loaded (/etc/systemd/system/boot-firewall.service; enabled; vendor preset: enabled)
   Active: active (exited) since Mon 2019-12-16 08:26:19 UTC; 5s ago
  Process: 7344 ExecStart=/usr/bin/boot-firewall (code=exited, status=0/SUCCESS)
 Main PID: 7344 (code=exited, status=0/SUCCESS)

Dec 16 08:26:19 router-fw systemd[1]: Starting iptables rules on boot...
Dec 16 08:26:19 router-fw boot-firewall[7344]: se han aplicado todas las reglas
Dec 16 08:26:19 router-fw systemd[1]: Started iptables rules on boot.
{% endhighlight %}

## MEJORA: Utiliza nuevas cadenas para clasificar el tráfico.

{% highlight bash %}
# Establecemos política ACCEPT para no perder la conexión al limpiar las tablas
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT

# Limpiamos las tablas
iptables -F
iptables -t nat -F
iptables -Z
iptables -t nat -Z

# [Borramos las cadenas]
# iptables -X ROUTER_A_DMZ
# iptables -X ROUTER_A_LAN
# iptables -X ROUTER_A_EXT
# iptables -X EXT_A_ROUTER
# iptables -X DMZ_A_ROUTER
# iptables -X DMZ_A_EXT
# iptables -X DMZ_A_LAN
# iptables -X EXT_A_DMZ
# iptables -X LAN_A_ROUTER
# iptables -X LAN_A_EXT
# iptables -X LAN_A_DMZ
# iptables -X EXT_A_LAN

# Creamos las nuevas cadenas

# Cadena router a DMZ
iptables -N ROUTER_A_DMZ
iptables -A OUTPUT -o eth2 -d 192.168.200.0/24 -j ROUTER_A_DMZ

# Cadena router a LAN
iptables -N ROUTER_A_LAN
iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -j ROUTER_A_LAN

# Cadena router al exterior
iptables -N ROUTER_A_EXT
iptables -A OUTPUT -o eth0 -j ROUTER_A_EXT

# Cadena exterior al router
iptables -N EXT_A_ROUTER
iptables -A INPUT -i eth0 -j EXT_A_ROUTER

# Cadena DMZ a router
iptables -N DMZ_A_ROUTER
iptables -A INPUT -i eth2 -s 192.168.200.0/24 -j DMZ_A_ROUTER

# Cadena DMZ al exterior
iptables -N DMZ_A_EXT
iptables -A FORWARD -i eth2 -o eth0 -s 192.168.200.0/24 -j DMZ_A_EXT

# Cadena DMZ a LAN
iptables -N DMZ_A_LAN
iptables -A FORWARD -i eth2 -o eth1 -s 192.168.200.0/24 -j DMZ_A_LAN

# Cadena EXT a DMZ
iptables -N EXT_A_DMZ
iptables -A FORWARD -i eth0 -o eth2 -j EXT_A_DMZ

# Cadena LAN a router
iptables -N LAN_A_ROUTER
iptables -A INPUT -i eth1 -s 192.168.100.0/24 -j LAN_A_ROUTER

# Cadena LAN al exterior
iptables -N LAN_A_EXT
iptables -A FORWARD -i eth1 -o eth0 -s 192.168.100.0/24 -j LAN_A_EXT

# Cadena LAN a DMZ
iptables -N LAN_A_DMZ
iptables -A FORWARD -i eth1 -o eth2 -s 192.168.100.0/24 -j LAN_A_DMZ

# Cadena EXT a LAN
iptables -N EXT_A_LAN
iptables -A FORWARD -i eth0 -o eth1 -j EXT_A_LAN



# Habilitamos ssh para administrar, vamos a utilizar el módulo para filtrar por MAC,
# utilizando la MAC de croqueta para que solo podamos administrar el router desde allí.
iptables -A INPUT -p tcp --dport 22 -m mac --mac-source fa:16:3e:8f:8e:6e -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT


# Establecemos la política
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

# Bit de forward
echo 1 > /proc/sys/net/ipv4/ip_forward


# Redirigimos el tráfico directo del puerto 2222 al 22
iptables -t nat -A PREROUTING -i eth0 -p tcp -m tcp --dport 2222 -j REDIRECT --to-ports 22


# Redirigimos el tráfico directo del puerto 22 a la loopback para inutilizarlo
iptables -t nat -A PREROUTING -i eth0 -p tcp -m tcp --dport 22 -j DNAT --to-destination 127.0.0.1:22


# Permitir SSH desde la LAN
iptables -A LAN_A_ROUTER -p tcp --dport 22 -j ACCEPT
iptables -A ROUTER_A_LAN -p tcp --sport 22 -j ACCEPT


# Permitir  SSH desde la DMZ
iptables -A DMZ_A_ROUTER -p tcp --dport 22 -j ACCEPT
iptables -A ROUTER_A_DMZ -p tcp --sport 22 -j ACCEPT



# Permitir tráfico loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT


# Permitir ping desde la DMZ
iptables -A DMZ_A_ROUTER -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A ROUTER_A_DMZ -p icmp -m icmp --icmp-type echo-reply -j ACCEPT


# Rechazar ping desde la LAN 
iptables -A LAN_A_ROUTER -p icmp -m icmp --icmp-type echo-request -j REJECT --reject-with icmp-port-unreachable


# Permitir ping a la LAN
iptables -A ROUTER_A_LAN -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A ROUTER_A_LAN -p icmp -m state --state RELATED -j ACCEPT
iptables -A ROUTER_A_LAN -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A LAN_A_ROUTER -p icmp -m icmp --icmp-type echo-reply -j ACCEPT


# Permitir ping a la DMZ
iptables -A ROUTER_A_DMZ -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A DMZ_A_ROUTER -p icmp -m icmp --icmp-type echo-reply -j ACCEPT


# Permitir ping al exterior
iptables -A ROUTER_A_EXT -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A EXT_A_ROUTER -p icmp -m icmp --icmp-type echo-reply -j ACCEPT


# Permitir ping y ssh desde la DMZ a la LAN
iptables -A DMZ_A_LAN -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A LAN_A_DMZ -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
iptables -A DMZ_A_LAN -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A LAN_A_DMZ -p icmp -m icmp --icmp-type echo-reply -j ACCEPT


# Permitir ssh desde la LAN a la DMZ
iptables -A LAN_A_DMZ -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A DMZ_A_LAN -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT


#Permitir acceso de DMZ y LAN al exterior
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.200.0/24 -o eth0 -j MASQUERADE


# Permitir ping desde la LAN al exterior
iptables -A LAN_A_EXT -p icmp -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A EXT_A_LAN -p icmp -m state --state ESTABLISHED -j ACCEPT


# Permitir navegación web desde la LAN
iptables -A LAN_A_EXT -p tcp -m multiport --dports 80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A EXT_A_LAN -p tcp -m multiport --sports 80,443 -m state --state ESTABLISHED -j ACCEPT
iptables -A LAN_A_EXT -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A EXT_A_LAN -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT


# Permitir navegación web desde la DMZ
iptables -A DMZ_A_EXT -p tcp -m multiport --dports 80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A EXT_A_DMZ -p tcp -m multiport --sports 80,443 -m state --state ESTABLISHED -j ACCEPT
iptables -A DMZ_A_EXT -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A EXT_A_DMZ -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT


# Permitir acceso al servidor WEB de la DMZ desde el exterior
iptables -A EXT_A_DMZ -p tcp -m multiport --dports 80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A DMZ_A_EXT -p tcp -m multiport --sports 80,443 -m state --state ESTABLISHED -j ACCEPT


# Permitir acceso al servidor FTP de la DMZ desde el exterior
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 21 -j DNAT --to 192.168.200.10:21
iptables -t nat -A POSTROUTING -o eth2 -p tcp --dport 21 -d 192.168.200.10 -j SNAT --to 192.168.200.2
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 20 -j DNAT --to 192.168.200.10:21
iptables -t nat -A POSTROUTING -o eth2 -p tcp --dport 20 -d 192.168.200.10 -j SNAT --to 192.168.200.2
iptables -A EXT_A_DMZ -p tcp --syn --dport 21 -m conntrack --ctstate NEW -j ACCEPT
iptables -A EXT_A_DMZ -i eth0 -o eth2 -p tcp --syn --dport 20 -m conntrack --ctstate NEW -j ACCEPT
iptables -A EXT_A_DMZ -i eth0 -o eth2 -p tcp -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A DMZ_A_EXT -i eth2 -o eth0 -p tcp -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT


# Permitir acceso al servidor WEB de la DMZ desde la LAN
iptables -A LAN_A_DMZ -p tcp -m multiport --dports 80,443 -d 192.168.200.10 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A DMZ_A_LAN -p tcp -m multiport --sports 80,443 -s 192.168.200.10 -m state --state ESTABLISHED -j ACCEPT


#Permitir acceso al servidor FTP de la DMZ desde la LAN
iptables -A LAN_A_DMZ -p tcp --syn --dport 21 -m conntrack --ctstate NEW -j ACCEPT
iptables -A LAN_A_DMZ -p tcp --syn --dport 20 -m conntrack --ctstate NEW -j ACCEPT
iptables -A LAN_A_DMZ -p tcp -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A DMZ_A_LAN -p tcp -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT


# Permitir acceso al servidor de correos de la DMZ desde la LAN
iptables -A LAN_A_DMZ -p tcp --dport 25 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A DMZ_A_LAN -p tcp --sport 25 -m state --state ESTABLISHED -j ACCEPT


# Permitir acceso al servidor MYSQL de la LAN desde la DMZ
iptables -A DMZ_A_LAN -p tcp --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A LAN_A_DMZ -p tcp --sport 3306 -m state --state ESTABLISHED -j ACCEPT
{% endhighlight %}

## MEJORA: Consruye el cortafuego utilizando nftables.

Utilizando la herramienta *iptables-translate*, y un pequeño _script_ para facilitar el trabajo, he transformado las reglas *iptables* que ya tenía creadas en el cortafuegos a *nftables*. El _script_ que he utilizado tiene el siguiente contenido:

{% highlight bash %}
#!/bin/sh

# Guarda todas las reglas iptables en un fichero,
# excepto las relgas de las directivas generales
iptables -S | grep -v '^\-P.*' > /home/debian/iptables.txt
iptables -t nat -S | grep -v '^\-P.*' >> /home/debian/iptables.txt

# Inyecta el contenido del fichero en la salida nº3
exec 3< /home/debian/iptables.txt
contador=0

# Recorre el fichero, y le da un formato correcto:
#       -Pone todo el tecto en minúsculas (con tr)
#       -Cambia el tipo "ip" a "innet" (con sed)
#       -Crea un fichero y añade las relgas
while read linea <&3; do
        if [ $contador -eq 0 ]; then
                iptables-translate $linea | tr [:upper:] [:lower:] \
                | sed 's/\ ip\ /\ inet\ /g' > nftables.txt
        else
                iptables-translate $linea | tr [:upper:] [:lower:] \
                | sed 's/\ ip\ /\ inet\ /g' >> nftables.txt
        fi
        contador=$(($contador+1))
done
{% endhighlight %}

Una vez ejecutado, me ha dado el siguiente resultado:

{% highlight bash %}
nft add rule inet filter input tcp dport 22 ether saddr fa:16:3e:8f:8e:6e counter accept
nft add rule inet filter input inet saddr 192.168.100.0/24 tcp dport 22 counter accept
nft add rule inet filter input inet saddr 192.168.200.0/24 tcp dport 22 counter accept
nft add rule inet filter input iifname "lo" counter accept
nft add rule inet filter input iifname "eth2" inet saddr 192.168.200.0/24 icmp type echo-request counter accept
nft add rule inet filter input iifname "eth1" inet saddr 192.168.100.0/24 icmp type echo-request counter reject
nft add rule inet filter input iifname "eth1" inet saddr 192.168.100.0/24 icmp type echo-reply counter accept
nft add rule inet filter input iifname "eth2" inet saddr 192.168.200.0/24 icmp type echo-reply counter accept
nft add rule inet filter input iifname "eth0" icmp type echo-reply counter accept
nft add rule inet filter output tcp sport 22 counter accept
nft add rule inet filter output inet daddr 192.168.100.0/24 tcp sport 22 counter accept
nft add rule inet filter output inet daddr 192.168.200.0/24 tcp sport 22 counter accept
nft add rule inet filter output oifname "lo" counter accept
nft add rule inet filter output oifname "eth2" inet daddr 192.168.200.0/24 icmp type echo-reply counter accept
nft add rule inet filter output oifname "eth1" inet daddr 192.168.100.0/24 icmp type echo-reply counter accept
nft add rule inet filter output oifname "eth1" inet protocol icmp inet daddr 192.168.100.0/24 ct state related  counter accept
nft add rule inet filter output oifname "eth1" inet daddr 192.168.100.0/24 icmp type echo-request counter accept
nft add rule inet filter output oifname "eth2" inet daddr 192.168.200.0/24 icmp type echo-request counter accept
nft add rule inet filter output oifname "eth0" icmp type echo-request counter accept
nft add rule inet filter forward iifname "eth2" oifname "eth1" tcp dport 22 ct state new,established  counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth2" tcp sport 22 ct state established  counter accept
nft add rule inet filter forward iifname "eth2" oifname "eth1" inet saddr 192.168.200.0/24 icmp type echo-request counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth2" inet daddr 192.168.200.0/24 icmp type echo-reply counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth2" tcp dport 22 ct state new,established  counter accept
nft add rule inet filter forward iifname "eth2" oifname "eth1" tcp sport 22 ct state established  counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth0" inet protocol icmp ct state new,established  counter accept
nft add rule inet filter forward iifname "eth0" oifname "eth1" inet protocol icmp ct state established  counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth0" inet protocol tcp tcp dport { 80,443} ct state new,established  counter accept
nft add rule inet filter forward iifname "eth0" oifname "eth1" inet protocol tcp tcp sport { 80,443} ct state established  counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth0" udp dport 53 ct state new,established  counter accept
nft add rule inet filter forward iifname "eth0" oifname "eth1" udp sport 53 ct state established  counter accept
nft add rule inet filter forward iifname "eth2" oifname "eth0" inet protocol tcp tcp dport { 80,443} ct state new,established  counter accept
nft add rule inet filter forward iifname "eth0" oifname "eth2" inet protocol tcp tcp sport { 80,443} ct state established  counter accept
nft add rule inet filter forward iifname "eth2" oifname "eth0" udp dport 53 ct state new,established  counter accept
nft add rule inet filter forward iifname "eth0" oifname "eth2" udp sport 53 ct state established  counter accept
nft add rule inet filter forward iifname "eth0" oifname "eth2" inet protocol tcp tcp dport { 80,443} ct state new,established  counter accept
nft add rule inet filter forward iifname "eth2" oifname "eth0" inet protocol tcp tcp sport { 80,443} ct state established  counter accept
nft add rule inet filter forward iifname "eth0" oifname "eth2" tcp dport 21 tcp flags & (fin|syn|rst|ack) == syn ct state new counter accept
nft add rule inet filter forward iifname "eth0" oifname "eth2" tcp dport 20 tcp flags & (fin|syn|rst|ack) == syn ct state new counter accept
nft add rule inet filter forward iifname "eth0" oifname "eth2" inet protocol tcp ct state related,established counter accept
nft add rule inet filter forward iifname "eth2" oifname "eth0" inet protocol tcp ct state related,established counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth2" inet protocol tcp inet daddr 192.168.200.10 tcp dport { 80,443} ct state new,established  counter accept
nft add rule inet filter forward iifname "eth2" oifname "eth1" inet protocol tcp inet saddr 192.168.200.10 tcp sport { 80,443} ct state established  counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth2" tcp dport 21 tcp flags & (fin|syn|rst|ack) == syn ct state new counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth2" tcp dport 20 tcp flags & (fin|syn|rst|ack) == syn ct state new counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth2" inet protocol tcp ct state related,established counter accept
nft add rule inet filter forward iifname "eth2" oifname "eth1" inet protocol tcp ct state related,established counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth2" tcp dport 25 ct state new,established  counter accept
nft add rule inet filter forward iifname "eth2" oifname "eth1" tcp sport 25 ct state established  counter accept
nft add rule inet filter forward iifname "eth2" oifname "eth1" tcp dport 3306 ct state new,established  counter accept
nft add rule inet filter forward iifname "eth1" oifname "eth2" tcp sport 3306 ct state established  counter accept
nft add rule inet filter prerouting iifname "eth0" tcp dport 2222 counter redirect to :22
nft add rule inet filter prerouting iifname "eth0" tcp dport 22 counter dnat to 127.0.0.1:22
nft add rule inet filter prerouting iifname "eth0" tcp dport 21 counter dnat to 192.168.200.10:21
nft add rule inet filter prerouting iifname "eth0" tcp dport 20 counter dnat to 192.168.200.10:21
nft add rule inet filter postrouting oifname "eth0" inet saddr 192.168.100.0/24 counter masquerade 
nft add rule inet filter postrouting oifname "eth0" inet saddr 192.168.200.0/24 counter masquerade 
nft add rule inet filter postrouting oifname "eth2" inet daddr 192.168.200.10 tcp dport 21 counter snat to 192.168.200.2
nft add rule inet filter postrouting oifname "eth2" inet daddr 192.168.200.10 tcp dport 20 counter snat to 192.168.200.2
{% endhighlight %}

Podríamos modificar el _script_ para que en vez de guardar las relgas en un fichero, limpiase las de *iptables* y ejecutase las de *nftables*. Esto sería un poco absurdo, ya que aunque introduzcamos las reglas con _comandos_ y _sintaxis_ de *iptables*, en realidad, se están ejecutando a bajo nivel reglas de *nftables*
