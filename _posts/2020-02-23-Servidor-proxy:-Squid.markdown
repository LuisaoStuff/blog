---
title:  "Servidor proxy: Squid"
excerpt: "Instalación y configuración del proxy Squid"
date:   2020-02-23 12:00:00
categories: [Servicios, Proxy]
---

## Introducción

Un _proxy_ es un servidor que ofrece, a **nivel de aplicación**, la posibilidad de conectar dos redes. Habitualmente se utiliza para hacer de intermediario entre una red local y el propio internet. Como conseguimos que **todo el tráfico** de nuestra red pase por **un solo nodo**, podemos aprovechar para aplicar **filtros**, situar una **caché** en esta máquina, etc.
Durante bastante tiempo los _proxys_ han generado bastante controversia, ya que coarta (en función del nivel de los filtros de búsquedas que apliquemos) la libertad de los usuarios, asmilándose al [mito de la caverna](https://es.wikipedia.org/wiki/Alegoría_de_la_caverna) de _Platón_.
No obstante en diversas ocasiones se ha utilizado principalmente como **caché**, cuyo fin era simplemente **mejorar** el **rendimiento** de la navegación web. Con la **proliferación** de las **páginas dinámicas**, cada vez **tiene menos sentido** la caché tanto de los **servidores proxy** como la de los **navegadores**.
En esta práctica vamos a _instalar_ y _configurar_ el paquete [Squid](http://www.squid-cache.org/) y aplicaremos algunas funcionalidades como **listas blancas**, **caché**, etc. Quien quiera reproducir el escenario, puede utilizar este fichero [Vagratfile](/docs/EscenarioProxy/Vagrantfile).

## Instalación

La instalación del paquete la llevaremos a cabo simplemente con el gestor de paquetes [apt](https://www.debian.org/doc/manuals/aptitude/pr01s03.es.html), por lo que ejecutamos:

{% highlight bash %}
apt install -y squid3
{% endhighlight %}

Después tendremos que **configurar** la ruta por defecto para que su **puerta de enlace** sea la dirección IP (de la red interna) del **servidor proxy**. En **cliente_int**, ejecutamos:

{% highlight bash %}
ip r del default
ip r add default via 10.0.0.10 dev eht1
{% endhighlight %}

**Squid** cuenta con un fichero de configuración bastante amplio (de 8563 lineas), pero para el ejercicio que estamos planteando necesitamos solo algunas de ellas. Dicho esto vamos a hacer una copia del original que llamaremos **squid.conf.old**, y seguidamente vaciaremos su contenido para añadir solo las siguientes lineas:

{% highlight bash %}
# Hacemos la copia de seguridad
cp /etc/squid/squid.conf /etc/squid/squid.conf.old
# Vaciamos el contenido del fichero efectivo
echo "" > /etc/squid/squid.conf
# Añadimos las siguientes lineas
nano /etc/squid/squid.conf

##############################################################################
acl localnet src 10.0.0.0/24            # Red interna de virtualbox (cliente_int)
acl localnet src 192.168.1.0/24         # Red del adaptador puente (Mi ordenador)

# Puertos permitidos
acl SSL_ports port 443		# https
acl Safe_ports port 80          # http
acl Safe_ports port 21          # ftp
acl Safe_ports port 1025-65535  # unregistered ports
acl CONNECT method CONNECT

http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access deny manager

include /etc/squid/conf.d/*

http_access allow localhost
http_access allow localnet

http_access deny all

http_port 8080

coredump_dir /var/spool/squid
##############################################################################
{% endhighlight %}


Con esta configuración estamos creando una [ACL](https://es.wikipedia.org/wiki/Lista_de_control_de_acceso) llamada **localnet**, que está definida por las dos redes definidas en el fichero **Vagrantfile**. De esta forma solo permitimos el acceso a través del proxy a las redes **10.0.0.0/24** y **192.168.1.0/24**, además de uso exclusivo de los puertos registrados **443**, **80**, y **21**.

### Uso del proxy desde la red 192.168.1.0/24

Para configurar el **proxy** en el cliente, podemos hacerlo de dos formas. Definiendo una **variable de entorno** en el sistema y especificando en el navegador que utilice los parámetros del sistema, o configurando directamente el **navegador**.

* **Variable de entorno**

Nos dirigimos a la ventana de preferencias de **Debian**, concretamente al apartado **Red**, configuramos el apartado de **proxy**, seleccionamos la opción manual e introducimos la dirección IP o el nombre de dominio del _servidor proxy_.

![](/images/EscenarioProxySQUID/debian-settings.png)

Después nos dirigimos al navegador (en mi caso **Mozilla Firefox**), al apartado de configuración. En la barra de búsqueda introducimos _"proxy"_.

![](/images/EscenarioProxySQUID/mozilla-settings1.png)

Y después seleccionamos **use system proxy settings**.

![](/images/EscenarioProxySQUID/mozilla-settings2.png)

Probamos con el paquete **wget** que funciona:

{% highlight bash %}
wget www.google.es
--2020-02-22 20:36:06--  http://www.google.es/
Conectando con 192.168.1.35:8080... conectado.
Petición Proxy enviada, esperando respuesta... 200 OK
Longitud: no especificado [text/html]
Grabando a: “index.html.1”

index.html.1            [ <=>                ]  12,63K  --.-KB/s    en 0,001s  

2020-02-22 20:36:06 (13,0 MB/s) - “index.html.1” guardado [12930]
{% endhighlight %}

* **Configuración en el navegador**

Simplemente nos dirigimos al mismo apartado de la configuración del navegador y seleccionamos **manual proxy configuration**:

![](/images/EscenarioProxySQUID/mozilla-settings3.png)

### Uso del proxy desde la red 10.0.0.0/24

Para configurar este cliente, lo hacemos exactamente igual que hicimos con el de la otra red. Solo que aquí al no disponer de entorno gráfico tendremos que definir la variable de entorno.

{% highlight bash %}
# http_proxy=http://IP-O-DOMINIO:PUERTO/
export http_proxy=http://10.0.0.10:8080/
{% endhighlight %}

Y probamos a acceder a **www.google.es**

{% highlight bash %}
wget www.google.es
--2020-02-22 19:23:22--  http://www.google.es/
Connecting to 10.0.0.10:8080... connected.
Proxy request sent, awaiting response... 200 OK
Length: unspecified [text/html]
Saving to: ‘index.html.1’

index.html.1            [ <=>                ]  12.63K  --.-KB/s    in 0s

2020-02-22 19:23:22 (55.3 MB/s) - ‘index.html.1’ saved [12933]
{% endhighlight %}

Si mostramos el fichero de accesos del _proxy_ **/var/log/squid/access.log** podemos observar que el _proxy_ está funcionando correctamente:

{% highlight bash %}
1582398719.974    541 10.0.0.11 TCP_MISS/301 776 GET http://www.linuxito.com/ - HIER_DIRECT/192.184.81.204 text/html
1582398726.015    200 10.0.0.11 TCP_MISS/200 13853 GET http://www.google.es/ - HIER_DIRECT/216.58.211.35 text/html
1582398733.620    205 10.0.0.11 TCP_MISS/304 370 GET http://deb.debian.org/debian/dists/buster/InRelease - HIER_DIRECT/151.101.134.133 -
1582398733.899    485 10.0.0.11 TCP_MISS/302 857 GET http://security.debian.org/debian-security/dists/buster/updates/InRelease - HIER_DIRECT/217.196.149.233 text/html
1582398734.030    106 10.0.0.11 TCP_MISS/304 410 GET http://security-cdn.debian.org/debian-security/dists/buster/updates/InRelease - HIER_DIRECT/151.101.132.204 -
1582398740.767    111 10.0.0.11 TCP_MISS/200 332424 GET http://deb.debian.org/debian/pool/main/c/curl/libcurl4_7.64.0-4_amd64.deb - HIER_DIRECT/151.101.134.133 application/x-debian-package
1582398741.024    256 10.0.0.11 TCP_MISS/200 265152 GET http://deb.debian.org/debian/pool/main/c/curl/curl_7.64.0-4_amd64.deb - HIER_DIRECT/151.101.134.133 application/x-debian-package
1582398830.441    151 192.168.1.49 TCP_MISS/200 915 POST http://ocsp.digicert.com/ - HIER_DIRECT/93.184.220.29 application/ocsp-response
1582398886.082    267 192.168.1.49 TCP_MISS/200 22390 GET http://images.linuxquestions.org/lqthumb.png - HIER_DIRECT/35.244.195.25 image/png
1582398886.889    175 192.168.1.49 TCP_MISS/200 4048 GET http://images.linuxquestions.org/favicon.ico - HIER_DIRECT/35.244.195.25 image/x-icon
1582399402.283    139 10.0.0.11 TCP_MISS/200 13815 GET http://www.google.es/ - HIER_DIRECT/216.58.211.35 text/html
1582400166.283    103 192.168.1.49 TCP_MISS/200 13824 GET http://www.google.es/ - HIER_DIRECT/216.58.211.35 text/html
{% endhighlight %}

## Listas Blancas y Negras

Para gestionar el acceso a internet utilizaremos listas blancas y negras. Una lista blanca se encarga de permitir **solo** el acceso a los dominios que figuran en ella y **deniegan** todo lo demás. Por el contrario, una lista negra **deniega** los dominios que contiene y **permite** el acceso a todo lo demás.

### Lista blanca

En el propio fichero de **/etc/squid/squid.conf** vamos a crear una **acl** con los siguientes parámetros.

{% highlight bash %}
acl whitelist dstdomain "/etc/squid/whitelist.txt"
http_access deny !whitelist
{% endhighlight %}

Es muy importante que esto lo definamos en las primeras lineas del fichero, puesto que, al igual que las reglas de **iptables**, _squid_ lo lee de forma **secuencial**.
Después creamos el fichero que hemos indicado y vamos añadiendo los dominios que queramos que se puedan acceder. En mi caso he añadido solo a www.google.com, por lo que solo debería poder acceder a esa página. Recargamos la configuración con:

{% highlight bash %}
squid -k reconfigure
{% endhighlight %}

Y probamos a acceder a dos páginas:

![](/images/EscenarioProxySQUID/whitelist.png)

### Lista Negra

De la misma forma, definimos otra acl (también situada en las primeras lineas del fichero), quedando así:

{% highlight bash %}
acl blacklist dstdomain "/etc/squid/blacklist.txt"
http_access deny blacklist
{% endhighlight %}

Igualmente creamos el correspondiente fichero y en mi caso solo pondré www.google.com. Recargamos la configuración y probamos.

![](/images/EscenarioProxySQUID/whitelist.png)

Como podemos observar, ahora no nos permite el acceso a www.google.com pero sí nos permite acceder a cualquier otra página.
