---
title:  "Introducción a Openshift con Minishift"
excerpt: "Jugando un poco con la herramienta de Minishift para desplegar nuestras primeras aplicaciones"
date:   2020-04-21  14:00:00
categories: [Contenedores, Docker, Kubernetes, OpenShift]
---

## Introducción

OpenShift es un producto que ofrece **plataforma como servicio** (o **PaaS**, en inglés _platform as a service_). Desarrollado por [Redhat](https://www.redhat.com/es) sobre **Kubernetes**, nos proporciona un sistema donde desplegar nuestras aplicaciones web, añadiendo funcionalidades adicionales a las que tenemos cuando utilizamos el "software base" de _Kubernetes_. Entre algunas de ellas tenemos [source2image](https://docs.openshift.com/enterprise/3.0/architecture/core_concepts/builds_and_image_streams.html#source-build) que se encargará de automatizar la creación de imágenes _docker_ basándose en el repositorio **github** donde tendremos nuestra aplicación.

Sin más preámbulo, empezaremos con la instalación de **Minishift**.

## Preparatorios

En esta ocasión usaremos la herramienta antes mencionada que nos proporcionará una instalación (de un solo nodo) de **OpenShift**. He de avisar que para hacer uso de esta aplicación, tendremos que tener un equipo con unos requisitos mínimos, debemos recordar que _OpenShift_ es una aplicación pensada para desplegarse en un **cluster**.
Podremos hacer uso de distintos sistemas de virtualización, pero en mi caso voy a utilizar **KVM** puesto que es el que más he usado durante mi formación y ha sido el "menos problemático".
Procedemos a instalar las dependencias:

{% highlight bash %}
apt update
# Instalamos las dependencias
apt install install qemu-kvm libvirt-daemon libvirt-daemon-system
# Añadimos a nuestro usuario al grupo libvirt
usermod -a -G libvirt $(whoami)
# Actualizamos nuestro grupo principal de forma temporal
newgroup libvirt
# Por último instalamos el driver correspondiente de KVM
curl -L https://github.com/dhiltgen/docker-machine-kvm/releases/download/v0.10.0/docker-machine-driver-kvm-ubuntu14.04 -o /usr/local/bin/docker-machine-driver-kvm
chmod +x /usr/local/bin/docker-machine-driver-kvm
{% endhighlight %}

Descargamos la versión correspondiente de **Minishift** desde el [repositorio oficial](https://github.com/minishift/minishift/releases), lo descomprimimos y movemos la aplicación a nuestro directorio donde se encuentran los demás binarios.

{% highlight bash %}
tar -xf minishift-1.34.2-linux-amd64.tgz
cd minishift-1.34.2-linux-amd64
cp minishift /usr/local/bin/
{% endhighlight %}

Después de todo esto ya podremos usar **minishift** como cualquier otro paquete que tengamos instalado. Por lo que ejecutamos:

{% highlight bash %}
minishift start
{% endhighlight %}

Este proceso tardará un rato, ya que tiene que descargar la imagen **ISO**, crear la máquina virtual, etc. Una vez que finalice el proceso obtendremos una salida como esta:

{% highlight bash %}
OpenShift server started.

The server is accessible via web console at:
    https://192.168.42.113:8443/console

You are logged in as:
    User:     developer
    Password: <any value>

To login as administrator:
    oc login -u system:admin
{% endhighlight %}

Como es una herramienta para desarrollar o simplemente experimentar un poco con OpenShift, obviamente carece de sistemas de seguridad, por lo que podremos acceder al panel web con el usuario _developer_ y **cualquier contraseña**.
Una vez que hayamos iniciado sesión, creamos un proyecto.

![](/images/intr-openshift/1.png)

Para empezar, vamos a desplegar simplemente un **servidor web** con una página estática **html**, y utilizaremos para ello el siguiente [repositorio](https://github.com/josedom24/html_for_openshift) junto con la imagen base de **php**.

![](/images/intr-openshift/2.png)

![](/images/intr-openshift/3.png)

A la hora de gestionar nuestros proyectos tenemos, a grandes rasgos, tres opciones:

* Panel web
* Uso del cliente **oc** a través de linea de comandos
* Crear un programa que haga uso de la **API**

## Panel web

Es el que tiene la funcionalidad más reducida de los tres, pero para empezar es el más intuitivo. En resumen vamos a probar las siguientes funciones:

* Balanceo de carga
* Rollback
* Despliegue continuo

### Balanceo de carga

A partir de un despliegue de dos **pods** o más, **OpenShift** se encargará de balancear la carga entre ellos. Si queremos aumentar el número de pods de nuestro proyect, tan solo tendremos que dirigirnos a la sección **Overview** (el panel general del proyecto) y pulsar sobre la flecha hacia arriba.

![](/images/intr-openshift/4.png)

Nosotros comprobaremos el balanceo de carga a través de un _script php_ con el siguiente contenido:

{% highlight php %}
<?php echo "Servidor:"; echo gethostname();echo "\n"; ?>
{% endhighlight %}

Básicamente lo que hace es imprimir por pantalla el nombre de la máquina donde se está ejecutando. Utilizaré un bucle for para realizar 10 peticiones con **curl** de ese fichero:

{% highlight bash %}
for i in {1..10}; do curl curso-openshift-curso-openshift.192.168.42.113.nip.io/info.php; doneServidor:curso-openshift-4-m7pnb
Servidor:curso-openshift-4-vcz2m
Servidor:curso-openshift-4-m7pnb
Servidor:curso-openshift-4-vcz2m
Servidor:curso-openshift-4-m7pnb
Servidor:curso-openshift-4-vcz2m
Servidor:curso-openshift-4-m7pnb
Servidor:curso-openshift-4-vcz2m
Servidor:curso-openshift-4-m7pnb
Servidor:curso-openshift-4-vcz2m
{% endhighlight %}

Y efectivamente podemos comprobar como se van alternando las máquinas en cada petición.

### Rollback

En este momento nos pondremos desde el punto de vista del **desarrollador**. En principio **no tendríamos que saber** como funciona todo esto por debajo, ya que para eso estamos utilizando esta **capa de abstracción** sobre **docker**, **kubernetes**, etc. Es por esto que a la hora de seguir un control de versiones, **OpenShift** nos facilita la vida.
Realizaremos un cambio del código en nuestro repositorio remoto, algo tan simple como cambiar una linea del **html**. Después solo tendremos que dirigirnos al apartado **builds**, entrar en nuestro proyecto y seleccionar **start build**.

![](/images/intr-openshift/5.png)

Si volvemos a acceder a la página podremos observar como se ha realizado el cambio una vez que se haya completado la "construcción" y el despliegue del nuevo **pod** con la nueva imagen:

![](/images/intr-openshift/6.png)

Si quisiéramos volver a una versión anterior por cualquier motivo, solo tenemos que acceder a la sección **deployments**, elegir el despliegue que queramos y pulsar sobre el botón **Roll back**:

![](/images/intr-openshift/7.png)

![](/images/intr-openshift/8.png)

### Despliegue Continuo

Como ya vimos en la entrada de integración continua con **jekyll** y **Travis CI**, **github** nos proporciona la posibilidad de hacer uso de **webhooks** para que aplicaciones como **OpenShift** realicen unas determinadas tareas en un determinado evento disparado por un _git commit_, un _git push_... en resumen, una modificación del repositorio.
Para activar esta funcionalidad, entramos en la configuración de los **builds** de nuestro proyecto y copiamos el enlace indicado como **Github Webhook URL**.

![](/images/intr-openshift/9.png)

Después nos dirigimos a la página de nuestro repositorio, accedemos a la configuración y seguidamente a **Webhooks**.

![](/images/intr-openshift/10.png)

Lo único que tendremos que hacer es pegar el enlace que hemos copiado antes e indicar que la información que se va a mandar es en formato **json**.

![](/images/intr-openshift/11.png)

A partir de ahora cada vez que hagamos un **git push** modificando este repositorio, se construirá automáticamente una nueva imagen y esta se desplegará.

## Cliente OC

La instalación del cliente es aún más sencilla que la de **minishift** ya que solo tendremos que descargar el paquete de la siguiente [página](https://github.com/openshift/origin/releases), descomprimirlo y mover el binario al _path_ correspondiente. En mi caso lo moveré a **/usr/local/bin**.
Una vez instalado, tendremos que iniciar sesión. La forma más sencilla es dirigirnos al panel web, hacer _click_ en nuestro usuario y seleccionamos **Copy login command**.
Después introducimos el comando en la terminal y deberíamos obtener una salida como esta:

{% highlight bash %}
oc login https://192.168.42.113:8443 --token=bxMkNw65lKmUyjTaY_r-8xgfCYAOR4w-jEc98NSy6gk
Logged into "https://192.168.42.113:8443" as "admin" using the token provided.

You have one project on this server: "curso-openshift"

Using project "curso-openshift".
{% endhighlight %}

Lo primero que haremos será borrar el proyecto anterior. Para ello introducimos la siguiente instrucción:

{% highlight bash %}
 oc delete project curso-openshift
project.project.openshift.io "curso-openshift" deleted
{% endhighlight %}

Acto seguido creamos uno nuevo:

{% highlight bash %}
oc new-project aplicacion-desde-oc
Now using project "aplicacion-desde-oc" on server "https://192.168.42.113:8443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-25-centos7~https://github.com/sclorg/ruby-ex.git

to build a new example application in Ruby.
{% endhighlight %}

Si queremos crear una nueva aplicación, tendremos que seguir la siguiente sintaxis:

{% highlight bash %}
oc new-app plantilla:versión~repositorioGitHub --name nombre-aplicación
{% endhighlight %}

En mi caso voy a lanzar de nuevo una aplicación usando el mismo repositorio y plantilla que antes:

{% highlight bash %}
oc new-app php:7.1~https://github.com/LuisaoStuff/html_for_openshift.git --name pagina-estatica

--> Found image dc5aa55 (4 months old) in image stream "openshift/php" under tag "7.1" for "php:7.1"

    Apache 2.4 with PHP 7.1 
    ----------------------- 
    PHP 7.1 available as container is a base platform for building and running various PHP 7.1 applications and frameworks. PHP is an HTML-embedded scripting language. PHP attempts to make it easy for developers to write dynamically generated web pages. PHP also offers built-in database integration for several commercial and non-commercial database management systems, so writing a database-enabled webpage with PHP is fairly simple. The most common use of PHP coding is probably as a replacement for CGI scripts.

    Tags: builder, php, php71, rh-php71

    * A source build using source code from https://github.com/LuisaoStuff/html_for_openshift.git will be created
      * The resulting image will be pushed to image stream tag "pagina-estatica:latest"
      * Use 'start-build' to trigger a new build
    * This image will be deployed in deployment config "pagina-estatica"
    * Ports 8080/tcp, 8443/tcp will be load balanced by service "pagina-estatica"
      * Other containers can access this service through the hostname "pagina-estatica"

--> Creating resources ...
    imagestream.image.openshift.io "pagina-estatica" created
    buildconfig.build.openshift.io "pagina-estatica" created
    deploymentconfig.apps.openshift.io "pagina-estatica" created
    service "pagina-estatica" created
--> Success
    Build scheduled, use 'oc logs -f bc/pagina-estatica' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/pagina-estatica' 
    Run 'oc status' to view your app.
{% endhighlight %}

Aunque aquí hemos creado la aplicación, deberemos dar un paso más y ejecutar `oc expose svc/pagina-estatica` para exponer el servicio y obtener una **URL** accesible. Después comprobamos el estado y vemos cuál es el link.

{% highlight bash %}
oc status
In project aplicacion-desde-oc on server https://192.168.42.113:8443

http://pagina-estatica-aplicacion-desde-oc.192.168.42.113.nip.io to pod port 8080-tcp (svc/pagina-estatica)
  dc/pagina-estatica deploys istag/pagina-estatica:latest <-
    bc/pagina-estatica source builds https://github.com/LuisaoStuff/html_for_openshift.git on openshift/php:7.1 
    deployment #1 deployed 4 minutes ago - 1 pod


2 infos identified, use 'oc status --suggest' to see details.
{% endhighlight %}


