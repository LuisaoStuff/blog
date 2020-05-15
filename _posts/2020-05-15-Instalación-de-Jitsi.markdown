---
title:  "Instalación de Jitsi"
excerpt: ""
date:   2020-05-15  14:00:00
categories: [Contenedores, Docker, Java]
---

## Introducción

En estos momentos de confinamiento y teletrabajo, la comunicación es un factor crucial y las videoconferencias están a la orden del día. En los últimos meses han surgido distintos casos en los que se **vulneraba la privacidad** de los usuarios en distintas plataformas como [Zoom](https://hipertextual.com/2020/04/zoom-expone-llamadas-internet).
Es más que probable que nos encontremos a día de hoy en la situación de que nuestra empresa necesite un servicio de estas carácterísticas, y por supuesto deberá ser seguro. En esta ocasión nos hemos decantado por [Jitsi](https://jitsi.org/the-mossos-confusion/) ya que se trata de un programa de código abierto y cuya instalación en **Debian** es relativamente sencilla, además de poder combinarlo con **Docker**.
Dicho esto, vamos a enseñaros dos posibles escenarios; instalación "clásica" e instalación con Docker.

## Instalación "clásica"

En este escenario, instalaremos Jitsi diréctamente en el servidor en cuestión. Empezaremos por añadir el repositorio de **jitsi** a nuestro fichero **/etc/apt/sources.list** y las claves correspondientes.

{% highlight bash %}
apt update && apt install gnupg
wget -qO - https://download.jitsi.org/jitsi-key.gpg.key | sudo apt-key add -
echo 'deb https://download.jitsi.org stable/' > /etc/apt/sources.list.d/jitsi-stable.list
sudo apt -y update
sudo apt -y install jitsi-meet
{% endhighlight %}

Durante la instalación nos pedirá el nombre de dominio que tendrá el sitio web donde se alojará **jitsi** y que el instalador usará para crear un **virtualhost** en **nginx** que funcionará como **proxy inverso**.

![](/images/jitsi/1.png)

También creará por el momento un certificado autofirmado, que podremos sustituir facilmente por uno de Let's Encrypt más tarde.
Como este escenario lo estoy desarrollando en una máquina virtual, estoy usando un nombre de dominio inventado (chat.jitsi.org), por lo que añadiré una entrada al fichero **/etc/hosts**.

Si probamos a acceder a dicho dominio, comprobamos que la página ya está operativa.

![](/images/jitsi/2.png)

## Instalación en contenedores Docker

Vamos a observar detenidamente el fichero del **Virtualhost** creado para **nginx**, de tal forma que podamos replicarlo y de esta forma separarlos en dos contenedores **Docker** distintos. A continuación voy a mostrar solo las partes más importantes, aunque podéis echarle un vistazo al fichero completo [aquí](/docs/jistsi/nginxFile.conf)

{% highlight bash %}
server_names_hash_bucket_size 64;

server {
    listen 80;
    listen [::]:80;
    server_name chat.jitsi.org;

    location ^~ /.well-known/acme-challenge/ {
       default_type "text/plain";
       root         /usr/share/jitsi-meet;
    }
    location = /.well-known/acme-challenge/ {
       return 404;
    }
    location / {
       return 301 https://$host$request_uri;
    }
}
server {
    listen 4444 ssl http2;
    listen [::]:4444 ssl http2;
    server_name chat.jitsi.org;
...

    root /usr/share/jitsi-meet;
...

    # BOSH
    location = /http-bind {
        proxy_pass      http://localhost:5280/http-bind;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $http_host;
    }

    # xmpp websockets
    location = /xmpp-websocket {
        proxy_pass http://127.0.0.1:5280/xmpp-websocket?prefix=$prefix&$args;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
        tcp_nodelay on;
    }
...
}
{% endhighlight %}

Podemos comprobar, que existen **3 proxys inversos**. Dos de ellos se encargan de "redirigir" las peticiones web (directivas **proxy_pass**) y el tercero se encarga de escuchar en el puerto 4444, que es uno de los que utilizará **jitsi** para los datos de video.
Jitsi tiene un repositorio en [github](https://github.com/jitsi/jitsi-meet/blob/master/doc/manual-install.md) donde explican como realizar la instalación de forma manual. En dicha explicación nos presentan un **esquema** de como funciona la **aplicación** completa (con el _proxy inverso_ incluido) a **nivel de red**.
> El siguiente esquema lo he hecho con la herramienta [visual-paradigm](https://online.visual-paradigm.com) basándome en el proporcionado por jitsi en su repositorio.

![](/images/jitsi/schema.jpg)
