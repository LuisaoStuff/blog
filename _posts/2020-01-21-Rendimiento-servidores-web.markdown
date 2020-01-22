---
title:  "Rendimiento de servidores web"
excerpt: "Comprobación de distintas configuraciones con php, python, apache y nginx"
date:   2020-01-20 10:59:00
categories: [Servicios]
tags: [apache,nginx,php,python]
---

## Introducción

A través de esta práctiva vamos a echarle un vistazo a las distintas configuraciones que nos permitirán la ejecución de código **php** y **python** en los _servidores web_ de **apache 2.4** y **nginx**, analizando en cada caso con cuál obtenemos un mayor rendimiento en base a un mayor número de peticiones concurrentes.


## Ejecución de scripts php

Para realizar esta tarea de la práctica vamos a preparar cinco configuraciones:

* Módulo php7-apache2
* PHP-FPM (socket unix) + apache2
* PHP-FPM (socket TCP) + apache2
* PHP-FPM (socket unix) + nginx
* PHP-FPM (socket TCP) + nginx

Vamos a instalar en la misma máquina los siguientes paquetes:

{% highlight bash %}
apt install mariadb-server apache2 nginx libapache2-mod-php php-fpm php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip
{% endhighlight %}

Descargamos e instalamos wordpress con su respectiva base de datos. Para modificar el tamaño máximo de los ficheros que podemos subir, nos tenemos que dirigir a */etc/php/7.3/apache2/php.ini* y modificar la siguiente lineas:

{% highlight ruby %}
...

; Maximum size of POST data that PHP will accept.
; Its value may be 0 to disable the limit. It is ignored if POST data reading
; is disabled through enable_post_data_reading.
; http://php.net/post-max-size
post_max_size = 512M

...

; Maximum allowed size for uploaded files.
; http://php.net/upload-max-filesize
upload_max_filesize = 512M

...
{% endhighlight %}

Tamaño cambiado:

!size-wp-mod.png!

Dependiendo de qué estemos utilizando para ejecutar el código php, la ruta del fichero *php.ini* será */etc/php/7.3/apache*, */etc/php/7.3/fpm*, etc.

### Módulo php7-apache2

* 10 peticiones concurrentes	:  100%    149 (longest request)
* 100 peticiones concurrentes	:  100%   1379 (longest request)
* 200 peticiones concurrentes	:  100%   7517 (longest request)
* 500 peticiones concurrentes	:  100%   9594 (longest request)
* 1000 peticiones concurrentes	:  100%   9164 (longest request)

### Apache2 con php-fpm socket unix

Para cambiar a php-fpm, tenemos que deshabilitar el módulo de *php7.3* y activar el módulo de *proxy_fcgi*. También tendremos que desactivar el módulo de *mpm_prefork* y activar *mpm_event*.

{% highlight bash %}
a2dismod php7.3
a2enmod proxy_fcgi
a2dismod mpm_prefork
a2enmod mpm_event
{% endhighlight %}

#### Pruebas

* 10 peticiones concurrentes	:  100%     66 (longest request)
* 100 peticiones concurrentes	:  100%    463 (longest request)
* 200 peticiones concurrentes	:  100%   4200 (longest request)
* 500 peticiones concurrentes	:  100%   7893 (longest request)
* 1000 peticiones concurrentes	:  100%   8829 (longest request)

### Apache2 con php-fpm socket tcp/ip

Para cambiar el socket a *tcp/ip* nos dirigimos primero al fichero */etc/php/7.3/fpm/pool.d/www.conf* y cambiamos la siguiente linea:

{% highlight bash %}
;listen = /run/php/php7.3-fpm.sock
listen = 127.0.0.1:9000
{% endhighlight %}

Y reiniciamos el servicio

{% highlight bash %}
systemctl restart php7.3-fpm
{% endhighlight %}

#### Pruebas

* 10 peticiones concurrentes	:  100%     55 (longest request)
* 100 peticiones concurrentes	:  100%    398 (longest request)
* 200 peticiones concurrentes	:  100%   2401 (longest request)
* 500 peticiones concurrentes	:  100%   9701 (longest request)
* 1000 peticiones concurrentes	:  100%   8847 (longest request)

### Nginx con php-fpm socket unix

Iniciamos nginx, y como tiene el mismo *documentroot* que apache, no necesitamos modificar demasiadas cosas en el fichero del *virtualhost*. Tan solo descomentaremos las lineas de fastcgi_pass:

{% highlight bash %}
location ~ \.php$ {
include snippets/fastcgi-php.conf;

# With php-fpm (or other unix sockets):
fastcgi_pass unix:/run/php/php7.3-fpm.sock;
# With php-cgi (or other tcp sockets):
#       fastcgi_pass 127.0.0.1:9000;
}
{% endhighlight %}

También tendremos que definir un nombre para el servidor, en mi caso lo llamaré *pruebawordpress.com*

#### Pruebas

* 10 peticiones concurrentes	:  100%     68 (longest request)
* 100 peticiones concurrentes	:  100%    362 (longest request)
* 200 peticiones concurrentes	:  100%   1660 (longest request)
* 500 peticiones concurrentes	:  100%   1740 (longest request)
* 1000 peticiones concurrentes	:  100%   3109 (longest request)

### Nginx con php-fpm socket tcp/ip

Cambiamos la configuración del *virtualhost* de _socket unix_ a _socket tcp/ip_:

{% highlight bash %}
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;

                # With php-fpm (or other unix sockets):
        #        fastcgi_pass unix:/run/php/php7.3-fpm.sock;
                # With php-cgi (or other tcp sockets):
               fastcgi_pass 127.0.0.1:9000;
        }
{% endhighlight %} 

Y de igual forma modificamos el fichero */etc/php/7.3/fpm/pool.d/www.conf* tal y como hicimos antes.

#### Pruebas

* 10 peticiones concurrentes	:  100%     66 (longest request)
* 100 peticiones concurrentes	:  100%    366 (longest request)
* 200 peticiones concurrentes	:  100%   9210 (longest request)
* 500 peticiones concurrentes	:  100%   8351 (longest request)
* 1000 peticiones concurrentes	:  100%   8680 (longest request)

## Aumento de rendimiento en la ejecución de scripts PHP

### Memcached

Como hemos podido comprobar hemos obtenido el mejor resultado con la combinación de **PHP-FPM** (**socket unix**) + **nginx**. No obstante, todavía podemos optimizar algo más el rendimiento utilizando el paquete **memcached**. Los paquetes a instalar son los siguientes:
{% highlight bash %}
apt install memcached php-memcached
{% endhighlight %} 

Después nos dirigimos al fichero de configuración ubicado en **/etc/memcached.conf** y cambiamos las siguientes lineas para que nos quede algo así:

{% highlight bash %} 
# memcached default config file
# 2003 - Jay Bonci <jaybonci@debian.org>
# This configuration file is read by the start-memcached script provided as
# part of the Debian GNU/Linux distribution.

# Run memcached as a daemon. This command is implied, and is not needed for the
# daemon to run. See the README.Debian that comes with this package for more
# information.
-d

# Log memcached's output to /var/log/memcached
logfile /var/log/memcached.log

# Be verbose
# -v

# Be even more verbose (print client commands as well)
# -vv

# Start with a cap of 64 megs of memory. It's reasonable, and the daemon default
# Note that the daemon will grow to this size, but does not start out holding this much
# memory
-m 64

# Default connection port is 11211
#-p 11211

# Run the daemon as root. The start-memcached will default to running as root if no
# -u command is present in this config file
-u memcache

# Specify which IP address to listen on. The default is to listen on all IP addresses
# This parameter is one of the only security measures that memcached has, so make sure
# it's listening on a firewalled interface.
#-l 127.0.0.1

#Añadimos el socket y los permisos

-s /var/run/memcached/memcached.sock
-a 775


# Limit the number of simultaneous incoming connections. The daemon default is 1024
# -c 1024

# Lock down all paged memory. Consult with the README and homepage before you do this
# -k

# Return error when memory is exhausted (rather than removing items)
# -M

# Maximize core file limit
# -r

# Use a pidfile
-P /var/run/memcached/memcached.pid
{% endhighlight %} 

Acto seguido modificamos al usuario **memcache** y lo añadimos al grupo **www-data** y reiniciamos el servicio

{% highlight bash %} 
 usermod -g www-data memcache
 systemctl restart memcached
{% endhighlight %} 

Y comprobamos que se ha creado el *socket* ejecutando <code>ls -l /var/run/memcached/</code>

{% highlight bash %}
$ ls -l /var/run/memcached/
total 4
-rw-r--r-- 1 memcache www-data 5 Jan 21 08:33 memcached.pid
srwxrwxr-x 1 memcache www-data 0 Jan 21 08:33 memcached.sock
{% endhighlight %}

#### Pruebas

* 10 peticiones concurrentes	:  100%     60 (longest request)
* 100 peticiones concurrentes	:  100%    384 (longest request)
* 200 peticiones concurrentes	:  100%    732 (longest request)
* 500 peticiones concurrentes	:  100%   1796 (longest request)
* 1000 peticiones concurrentes	:  100%   1918 (longest request)

Como podemos observar hemos obtenido una mejora significativa en el rendimiento. Sin *memcached*, en las 1000 peticiones concurrentes obtuvimos *3109*, mientras que con *memcached* instalado hemos rebajado hasta *1918*, casi la mitad!

### Varnish

Otro método para mejorar *nginx* es el uso del **proxy inverso** *Varnish*. Primero instalamos el paquete con <code>apt</code> y luego nos dirigimos a los ficheros de configuración. Abrimos el fichero **/etc/varnish/default.vcl** y modificamos los siguientes apartados para que queden así:

{% highlight bash %}
sub vcl_recv {
    unset req.http.cookie;
}
sub vcl_fetch {
    unset beresp.http.set-cookie;
}
{% endhighlight %}

Despues modificamos el fichero **/etc/default/varnish**, cambiando la el parámetro **-a** de la directiva *DAEMON_OPTS* para que escuche por el puerto **80**, quedando de la siguiente forma:

{% highlight bash %}
DAEMON_OPTS="-a :80
             -T localhost:6082
             -f /etc/varnish/default.vcl
             -S /etc/varnish/secret
             -s malloc,256M"
{% endhighlight %}

Además tendremos que modificar el fichero de configuración de la unidad de *systemd* para que cambie definitivamente el puerto. La unidad de systemd está ubicada en **/lib/systemd/system/varnish.service** y tendremos que modificar la directiva **ExecStart** para que quede algo así:

{% highlight bash %}
ExecStart=/usr/sbin/varnishd -j unix,user=vcache -F -a :80 -T localhost:6082 -f /etc/varnish/default.vcl -S /etc/varnish/secret -s malloc,256m
{% endhighlight %}

Por último cambiamos el puerto de escucha de nginx al *8080*. Para ello cambiamos el parámetro *listen* del fichero de configuración del *virtualhost* y luego reiniciamos ambos servicios.

{% highlight bash %}
...
        listen 8080;
        listen [::]:8080;
...
{% endhighlight %}

#### Pruebas

* 10 peticiones concurrentes	:  100%     25 (longest request)
* 100 peticiones concurrentes	:  100%     70 (longest request)
* 200 peticiones concurrentes	:  100%   1048 (longest request)
* 500 peticiones concurrentes	:  100%   1098 (longest request)
* 1000 peticiones concurrentes	:  100%   1134 (longest request)

<a href="/images/bestphp.png"><img src="/images/bestphp.png" /></a>

## Ejecución de scripts Python

En esta parte de la práctica veremos las siguientes combinaciones:

* apache2 + Módulo wsgi
* apache2 + gunicorn
* apache2 + uwsgi
* nginx + gunicorn
* nginx + uwsgi

Para probar el rendimiento, vamos a instalar el _cms mezzanine_. Primero tenemos que instalar **git** ya que vamos a usar una plantilla y por lo tanto vamos a clonar un repositorio.
{% highlight bash %}
apt install git
mkdir /var/www/mezzanine
cd /var/www/mezzanine
git clone https://github.com/thecodinghouse/mezzanine-themes.git
{% endhighlight %}
Después instalamos **pip**, el gestor de paquetes de **python** y activaremos el módulo de **wsgi** en _apache_.
{% highlight bash %}
apt install python3-pip
pip install mezzanine

python manage.py createdb
{% endhighlight %}


