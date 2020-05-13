---
title:  "Despliegue de una aplicación Python con Docker"
excerpt: "Esta vez desplegaremos un cms python"
date:   2020-04-25  14:00:00
categories: [Contenedores, Docker, Python]
---

## Introducción

En esta entrada aprenderemos tanto a crear una **imagen docker** para desplegar la aplicación **pyhton** como a usar la imagen oficial de los repositorios. También haremos uso de **docker compose** para el despliegue de varios contenedores en el caso de que queramos implantar una aplicación, como un cms, que requiera de múltiples nodos. Ya sea para utilizar una **base de datos**, un servidor de aplicaciones (como **gunicorn**), etc.

## Imagen docker "a mano"

Antes de nada, primero tendremos que instalar el paquete **docker.io** con el gestor de paquetes **apt**. Dicho esto, la aplicación que queremos implantar es una que utilizó mi instituto en su momento para la gestión de proyectos y de los distintos módulos de los alumnos. Es por esto que lo primero será clonar el repositorio **git**.

{% highlight bash %}
git clone https://github.com/jd-iesgn/iaw_gestionGN.git
{% endhighlight %}

Tendremos que modificar el fichero de configuración para que la aplicación funcione utilizando una base de datos **mysql**, en lugar de **sqlite**. Concretamente este fichero es **gestion/settings.py**, y modificaremos dos secciones.

{% highlight python %}
ALLOWED_HOSTS = ['www.gestiona.com']

...

DATABASES = {
      'default': {
          'ENGINE': 'mysql.connector.django',
          'NAME': 'iesgn',
          'USER': 'iesgn',
          'PASSWORD': 'dios',
          'HOST': 'mysqlDB',
          'PORT': '',
      }
  }
{% endhighlight %}

#### Dominio

* **ALLOWED_HOSTS**: Es el dominio que utilizaremos para acceder a la aplicación (y que estableceremos en el fichero **/etc/hosts**)

#### Base de datos

* **'ENGINE': 'mysql.connector.django'**: Es el tipo de conector **mysql** que usará python.
* **HOST**: Aquí estableceremos el nombre del contenedor docker que tendrá la base de datos.

Como vamos a utilizar un contenedor a parte para alojar la base de datos, vamos a dejarlo ya preparado. Empezaremos **definiendo la red** con la que se comunicarán ambos contenedores y que llamaremos "instituto".

{% highlight bash %}
docker network create instituto
{% endhighlight %}

Después utilizando la imagen docker oficial de mysql, crearemos el contenedor con los siguientes parámetros:

{% highlight bash %}
docker run -d --name mysqlDB --network instituto -v /opt/bbdd_mariadb:/var/lib/mysql -e MYSQL_DATABASE=iesgn -e MYSQL_USER=iesgn -e MYSQL_PASSWORD=dios -e MYSQL_ROOT_PASSWORD=dios mariadb
{% endhighlight %}

Dichos parámetros tienen que coincidir con los que definimos en el fichero **gestion/settings.py** que modificamos antes. Además de dejar preestablecido el nombre de usuario, el nombre de la base de datos y la contraseña, he definido (con el parámetro **-v /opt/bbdd_mariadb:/var/lib/mysql**) la ruta local su semejante dentro del contenedor del directorio donde se ubica la base de datos para que estos sean persistentes.

Una vez hemos dejado preparada la base de datos, podemos centrarnos con el contenedor de nuestra aplicación. El contenido del **Dockerfile**, cuya **imagen base** será la de **Debian**, será el siguiente:

{% highlight bash %}
FROM debian
RUN apt update
RUN apt install -y apache2 libapache2-mod-wsgi-py3 python3-pip python3-mysqldb \
zlib1g-dev libjpeg62-turbo-dev
RUN apt clean && rm -rf /var/lib/apt/lists/*

RUN pip3 install mysql-connector-python
EXPOSE 80

COPY ./iaw_gestionGN /var/www/iaw_gestionGN
RUN pip3 install -r /var/www/iaw_gestionGN/requirements.txt
COPY ./000-default.conf /etc/apache2/sites-available
RUN cp -r /usr/local/lib/python3.7/dist-packages/django/contrib/admin/static/admin/ \
/var/www/iaw_gestionGN/static
RUN chown -R www-data: /var/www/iaw_gestionGN
ENTRYPOINT ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
{% endhighlight %}

Entre las distintas instrucciones que hemos indicado en el **Dockerfile**, hay una que espeficica la copia del fichero de configuración del **virtualhost** (_000-default.conf_) en la ruta correspondiente de **apache**, por lo que tendremos que dejarlo preparado también. Tendrá el siguiente contenido:

{% highlight bash %}
<VirtualHost *:80>
    DocumentRoot /var/www/iaw_gestionGN
    ServerName www.gestiona.com
    WSGIDaemonProcess iaw_gestionGN user=www-data group=www-data processes=1 threads=5 python-path=/var/www/iaw_gestionGN
    WSGIScriptAlias / /var/www/iaw_gestionGN/gestion/wsgi.py

    <Directory /var/www/iaw_gestionGN>
            WSGIProcessGroup iaw_gestionGN
            WSGIApplicationGroup %{GLOBAL}
            Require all granted
    </Directory>
    Alias "/static/" "/var/www/iaw_gestionGN/static/"
</VirtualHost>
{% endhighlight %}

Una vez hecho esto, ya podemos construir la imagen.

{% highlight bash %}
docker build luisvazquezalejo/iaw_gestiona:v1 .
{% endhighlight %}

Después lanzamos el contenedor:

{% highlight bash %}
docker run -d --name aplicacion --network instituto -p 80:80 luisvazquezalejo/iaw_gestiona:v1
{% endhighlight %}

Y por último ejecutamos la siguiente instrucción para inicializar la base de datos y añadir los datos del fichero **datos.json**

{% highlight bash %}
docker exec aplicacion python3 /var/www/iaw_gestionGN/manage.py loaddata /var/www/iaw_gestionGN/datos.json
docker exec aplicacion python3 /var/www/iaw_gestionGN/manage.py migrate
{% endhighlight %}

Probamos a acceder a la aplicación:

![](/images/docker-python-28/1.png)

![](/images/docker-python-28/2.png)

## Imagen docker desde DockerHub

Ahora en vez de usar la imagen base de Debian, nos limitaremos a utilizar la imagen proporcionada por **Dockerhub** de **python**. Esto facilita bastante las cosas y además de tener un **Dockerfile** bastante más corto.

{% highlight bash %}
FROM python
WORKDIR /usr/src/app
RUN pip3 install mysql-connector-python
COPY ./iaw_gestionGN/requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY ./iaw_gestionGN .
EXPOSE 8080
CMD ["python", "manage.py", "collectstatic"]
CMD ["python", "manage.py", "runserver", "0.0.0.0:80"]
{% endhighlight %}

Construimos la imagen:

{% highlight bash %}
docker build luisvazquezalejo/iaw_gestiona:v2 .
{% endhighlight %}

Para levantar tanto el contenedor de la aplicación como el de la base de datos, utilizaremos **Docker Compose**, por lo que primero, tendremos que instalar el paquete:

{% highlight bash %}
apt install docker-compose
{% endhighlight %}

Y creamos el fichero **docker-compose.yml**.


{% highlight yaml %}
version: '3.1'

services:
  mysqlDB:
    container_name: mysqlDB
    image: mariadb
    restart: always
    environment:
      MYSQL_DATABASE: iesgn
      MYSQL_USER: iesgn
      MYSQL_PASSWORD: dios
      MYSQL_ROOT_PASSWORD: dios
    volumes:
      - /opt/bbdd_mariadb:/var/lib/mysql
  aplicacion:
    container_name: aplicacion
    image: luisvazquezalejo/iaw_gestiona:v2
    restart: always
    depends_on:
      - mysqlDB
    ports:
      - 80:80
{% endhighlight %}

Levantamos el escenario ejecutanto `docker-compose up -d`, y más tarde introducimos el mismo comando que antes para inicializar la base de datos, solo que modificando el directorio del fichero **manage.py**.

{% highlight bash %}
docker exec aplicacion python3 /usr/src/app/manage.py migrate
{% endhighlight %}

## Ejecución de python a través de Gunicorn

En este caso desplegaremos un total de **3 contenedores**; el primero tendrá el **servidor web**, el segundo la base de datos **mysql** y el tercero el **servidor de aplicaciones** [gunicorn](https://gunicorn.org/).
Empezaremos por definir una nueva imagen con la que construiremos el contenedor del servidor web. Dicha imagen tendrá el siguiente contenido:

{% highlight bash %}
FROM nginx
WORKDIR /var/www/iaw_gestionGN/
RUN apt update && apt install -y python3-pip python3-mysqldb zlib1g-dev libjpeg62-turbo-dev 
RUN apt clean && rm -rf /var/lib/apt/lists/*
RUN pip3 install mysql-connector-python
COPY ./iaw_gestionGN/requirements.txt ./
RUN pip3 install -r requirements.txt
EXPOSE 80
CMD ["python3", "manage.py", "collectstatic"]
CMD ["nginx", "-g", "daemon off;"]
{% endhighlight %}

A continuación vamos a definir también el _Dockerfile_ para que tenga **uwsgi** instalado y todas las dependencias de nuestra aplicación.

{% highlight bash %}
FROM debian
WORKDIR /var/www/html
RUN apt update
RUN apt install -y zlib1g-dev libjpeg62-turbo-dev uwsgi uwsgi-plugin-python3 python3-pip python3-mysqldb
RUN apt clean && rm -rf /var/lib/apt/lists/*
RUN pip3 install mysql-connector-python
COPY ./iaw_gestionGN/requirements.txt ./
RUN pip3 install -r requirements.txt
EXPOSE 8080
{% endhighlight %}

También tendremos que configurar el fichero **default.conf** de nginx, de tal forma que actúe como _proxy inverso_ del servidor de aplicaciones **uwsgi**.

{% highlight bash %}
server {
   listen 80;
   server_name www.gestiona.com;

    # Add headers to serve security related headers
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header X-Download-Options noopen;
    add_header X-Permitted-Cross-Domain-Policies none;
    add_header Referrer-Policy no-referrer;

    root /var/www/iaw_gestionGN/;
    location / {
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header Host $http_host;
      proxy_redirect off;
      proxy_pass http://uwsgi:8080;
    }
    location /static/admin/css {
        alias /usr/local/lib/python3.7/dist-packages/django/contrib/admin/static/admin/css;
    }
    location /static {
        alias /var/www/iaw_gestionGN/static;
    }
    location /media {
        alias /var/www/iaw_gestionGN/media;
    }
}
{% endhighlight %}

Como estamos usando **docker**, la directiva **proxy_pass** la definimos en base al nombre del contenedor y al puerto por el que se comunica. Además de la misma forma que en otros escenarios hemos tenido que definir algunos **alias** en consecuencia de las rutas del **css**, este no va a ser una escepción. De modo que además de establecer las directivas necesarias para el **proxy inverso**, también definiremos dos _redirecciones internas_ para que funcione la hoja de estilo (directivas **location y **alias**).

Por último creamos el script **docker-compose** para levantar los tres contenedores a la vez:

{% highlight yaml %}
version: '3.1'
services:
  mysqlDB:
    container_name: mysqlDB
    image: mariadb
    restart: always
    environment:
      MYSQL_DATABASE:
      MYSQL_USER: iesgn
      MYSQL_PASSWORD: dios
      MYSQL_ROOT_PASSWORD: dios
    volumes:
      - /opt/bbdd_mariadb:/var/lib/mysql
  uwsgi:
    container_name: uwsgi
    image: luisvazquezalejo/uwsgiserver:v1
    restart: always
    volumes:
      - ./iaw_gestionGN:/var/www/iaw_gestionGN
    command: uwsgi --http-socket :8080 --plugin python37 --chdir /var/www/iaw_gestionGN --wsgi-file gestion/wsgi.py --process 4 --threads 2 --master
  nginx:
    container_name: nginx
    image: luisvazquezalejo/iaw_gestion:v3
    restart: always
    depends_on:
      - mysqlDB
      - uwsgi
    ports:
      - 80:80
    volumes:
    - ./iaw_gestionGN:/var/www/iaw_gestionGN
    - ./default.conf:/etc/nginx/conf.d/default.conf
{% endhighlight %}

Ejecutamos la orden `docker-compose up -d` y observamos como todos los contenedores están funcionando:

{% highlight bash %}
Creating mysqlDB ... done
Creating uwsgi   ... done
Creating nginx   ... done

$~ docker ps

CONTAINER ID        IMAGE                             COMMAND                  CREATED             STATUS              PORTS                NAMES
984a98daaea8        luisvazquezalejo/iaw_gestion:v3   "nginx -g 'daemon of…"   10 minutes ago      Up 9 minutes        0.0.0.0:80->80/tcp   nginx
8cf41c1019d9        luisvazquezalejo/uwsgiserver:v1   "uwsgi --http-socket…"   10 minutes ago      Up 10 minutes       8080/tcp             uwsgi
5c01f49a17da        mariadb                           "docker-entrypoint.s…"   10 minutes ago      Up 10 minutes       3306/tcp             mysqlDB
{% endhighlight %}

## Django CMS en Docker

Probaremos ahora la instalación sencilla de [Django CMS](https://www.django-cms.org/en/) a partir de la imagen base de **python**. Por lo que el **Dockerfile** debería ser algo como esto:

{% highlight bash %}
FROM python:3
WORKDIR /usr/src/app
RUN apt update 
RUN apt install -y python3-mysqldb zlib1g-dev libjpeg62-turbo-dev 
RUN apt clean && rm -rf /var/lib/apt/lists/*
RUN pip3 install mysql-connector-python
RUN pip3 install djangocms-installer
EXPOSE 80
RUN djangocms mysite
RUN pip3 install -r /usr/src/app/mysite/requirements.txt
COPY ./script.bash /tmp
RUN chmod +x /tmp/script.bash
RUN /tmp/script.bash
CMD ["python3", "/usr/src/app/mysite/manage.py", "migrate"]
CMD ["python3", "/usr/src/app/mysite/manage.py", "runserver", "0.0.0.0:80"]
{% endhighlight %}

He añadido un **script** escrito en **bash** para poder modificar un par de lineas en el _fichero de configuración_ del **CMS** de tal forma que podamos acceder con el nombre de dominio **www.pythonCMS.com** que añadiremos al fichero **/etc/hosts**. El _script_ en sí es bastante sencillo, solo contiene estas dos lineas más la cabecera.

{% highlight bash %}
#!/bin/bash
sed -i 's/project.db/mysite\/project.db/g' /usr/src/app/mysite/mysite/settings.py
sed -i 's/ALLOWED_HOSTS = \[\]/ALLOWED_HOSTS = \["www.pythonCMS.com"\]/g' /usr/src/app/mysite/mysite/settings.py
{% endhighlight %}

Ya solo nos queda construir la imagen y lanzar el contenedor.

{% highlight bash %}
docker build -t luisvazquezalejo/djangocms:v1 .
docker run -d --name aplicacion -p 80:80 luisvazquezalejo/djangocms:v1
{% endhighlight %}

Probamos a acceder con el dominio que establecimos antes:

![](/images/docker-python-28/3.png)

![](/images/docker-python-28/4.png)

## DjangoCMS + Postgresql

En este caso vamos a separar la base de datos y la aplicación en dos contenedores. Para esto seguiremos la documentación que nos ofrece [DjangoCMS](https://docs.django-cms.org/en/release-3.4.x/introduction/install.html) y definiremos en el fichero **settings.py** los parámetros correspodientes, como nombre de la base de datos, usuario, etc. El _script_ ejecutará una serie de instrucciones con **sed** para realizar las **modificaciones** de forma **no interactiva**.

{% highlight bash %}
#!/bin/bash
sed -i 's/project.db/django/g' /usr/src/app/mysite/mysite/settings.py
sed -i 's/ALLOWED_HOSTS = \[\]/ALLOWED_HOSTS = \["www.pythonCMS.com"\]/g' /usr/src/app/mysite/mysite/settings.py
sed -i 's/django.db.backends.sqlite3/django.db.backends.postgres/g' /usr/src/app/mysite/mysite/settings.py
sed -i 's/localhost/postgresDB/g' /usr/src/app/mysite/mysite/settings.py
sed -i "s/'PASSWORD': '',/'PASSWORD': 'admin',/g" /usr/src/app/mysite/mysite/settings.py
sed -i "s/'USER': ''/'USER': 'admin'/g" /usr/src/app/mysite/mysite/settings.py
{% endhighlight %}

Como estamos usando ahora una base de datos **Postgresql**, tendremos que instalar con **pip** el paquete correspondiente para que la aplicación pueda comunicarse con dicha base de datos. De esta forma tendremos que modificar el fichero **Dockerfile** de la siguiente forma:

{% highlight bash %}
FROM python:3
WORKDIR /usr/src/app
RUN apt update 
RUN apt install -y zlib1g-dev libjpeg62-turbo-dev
RUN apt clean && rm -rf /var/lib/apt/lists/*
RUN pip3 install djangocms-installer
EXPOSE 80
RUN djangocms mysite
RUN pip3 install -r /usr/src/app/mysite/requirements.txt
RUN pip3 intall psycopg2
COPY ./script.bash /tmp
RUN chmod +x /tmp/script.bash
RUN /tmp/script.bash
CMD ["python3", "/usr/src/app/mysite/manage.py", "migrate"]
CMD ["python3", "/usr/src/app/mysite/manage.py", "runserver", "0.0.0.0:80"]
{% endhighlight %}

Creamos la red, lanzamos el contenedor con la **base de datos** indicando el **usuario admin** y seguidamente lanzamos el contenedor con la aplicación:

{% highlight bash %}
docker network create cms

docker run -d --name postgresDB --network cms \
-e POSTGRES_DB=django -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=admin \
-e PGDATA=/var/lib/postgresql/data/pgdata \
-v /opt/bbdd_postgres:/var/lib/postgresql/data postgres

docker run -d --name aplicacion --network cms -p 80:80 luisvazquezalejo/djangocms:v1
{% endhighlight %}

Además la primera vez que lancemos los contenedores, tendremos que ejecutar dos instrucciones para que cree las tablas en la base de datos y el usuario admin. De tal modo que ejecutamos una consola bash dentro del contenedor del contenedor **aplicacion** e introducimos las siguientes instrucciones:


{% highlight bash %}
docker exec -it aplicacion /bin/bash
root@30013efdd826:/usr/src/app# python3 mysite/manage.py migrate
root@30013efdd826:/usr/src/app# python3 mysite/manage.py createsuperuser
{% endhighlight %}

Probamos a acceder a la aplicación y añadir una página.

![](/images/docker-python-28/5.png)

Comprobamos como se está generando contenido en el directorio **/opt/bbdd_postgres/pgdata/**.

{% highlight bash %}
ls /opt/bbdd_postgres/pgdata/
base	      pg_ident.conf  pg_serial	   pg_tblspc	postgresql.auto.conf
global	      pg_logical     pg_snapshots  pg_twophase	postgresql.conf
pg_commit_ts  pg_multixact   pg_stat	   PG_VERSION	postmaster.opts
pg_dynshmem   pg_notify      pg_stat_tmp   pg_wal	postmaster.pid
pg_hba.conf   pg_replslot    pg_subtrans   pg_xact
{% endhighlight %}

También podemos comprobar que está definido el uso de la base de datos **postgres** correctamente si accedemos al contenedor y vemos la sección **DATABASES** del fichero **settings.py**.

{% highlight python %}
DATABASES = {
    'default': {
        'CONN_MAX_AGE': 0,
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'postgresDB',
        'NAME': 'django',
        'PASSWORD': 'admin',
        'PORT': '',
        'USER': 'admin'
    }
}
{% endhighlight %}

