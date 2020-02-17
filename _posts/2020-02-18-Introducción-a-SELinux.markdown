---
title:  "Introducción a SELinux"
excerpt: "Conceptos básicos y funcionamiento de SELinux en Centos 8"
date:   2020-02-18 14:25:00
categories: [Sistemas, Seguridad]
---


## Introducción

**SELinux**, para quien no lo conozca es un sistema de seguridad basado en el otorgamiento de permisos a usuarios y demonios sobre ficheros, directorios, protocolos, etc. Bajo esta premisa se establecen una serie de reglas por defecto que prohíben a cualquier proceso, acceder o utilizar según que protocolo que esté fuera de su configuración estandar o por defecto. Es por esto que la **filosofía** que debería seguir todo administrador encargado de gestionar un sistema con **SELinux** (en el modo _enforcing_) debería ser, **utilizar** en la medidida de lo posible únicamente los **directorios de trabajo por defecto** en cada aplicación.
Existen infinidad de casos. Uno por ejemplo sería utilizar el _DocumentRoot_ de los _virtualhost_ de **Nginx** en **/usr/share/** y no en **/var/www/** como acostumbramos a usar con **apache2**.
No obstante, existen numerosas circustancias en las que nos vemos obligados a cambiar dichos directorios de trabajo, ya sea por facilitar tareas como las copias de seguridad o por políticas de la empresa. A continuación vamos a ver las distintas **herramientas** que nos ofrece **SELinux** para añadir y modificar reglas.

## Funcionamiento

Durante varios años, cuando se estaba empezando a desarrollar el proyecto del **Kernel Linux**, se estuvo debatiendo sobre si se debería cambiar el modelo de **Kernel Monolítico** a **Micro-Kernel**. Los dos presentaban sus ventajas pero acabó decidiéndose mantener el Kernel Monolítico. Debido a esto no es ninguna locura pensar que una apliación pueda conseguir permisos a nivel de _Kernel_ (eso si, dependiendo de las [kernel capabilities](https://www.incibe-cert.es/blog/linux-capabilities) que tenga el proceso).
**SELinux** es una aplicación que funciona a nivel de **Kernel**, controlando el acceso de las aplicaciones a según que objetos del sistema.
Para poder aplicar las reglas que definirán las políticas de acceso, se utilizan los contextos y las normas de carácter _booleano_. 

### Modos de funcionamiento

**SELinux** presenta tres modos de funcionamiento principales; _enforcing_, _permissive_ y _disabled_. Desafortunadamente existe demasiada gente que ya sea por la falta de formación en **SELinux** o simple pereza, lo primero que hacen es desactivarlo nada más instalar **RHEL** o **Centos**. En esta entrada no vamos a tener en cuenta esta última opción (disabled), y vamos a centrarnos en las dos primeras.

* **enforcing mode**: En este modo, **denegará** cualquier acción sobre cualquier objeto que no esté definida previamente en la política. Este es el modo que viene activado por defecto.

* **permisive mode**: Se **permiten** todas las **acciones**, pero aquellas que incumplan la política definida, serán **notificadas** en el **log** (/var/log/messages).

Para consultar el estado actual de SELinux, utilizamos el comando `getenforce` o `sestatus` si queremos una salida más detallada.

{% highlight bash %}
[root@salmorejo centos]# getenforce
Enforcing

[root@salmorejo centos]# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      31
{% endhighlight %}

Si queremos cambiar entre los modos de SELinux, utilizamos `setenforce Enforcing | Permissive`.

## Boolean SELinux

las normas de carácter _booleano_ son bastante fáciles de entender, y son simplemente un conjunto de reglas verdadero/falso, que determinan el acceso de una aplicación a un protocolo. Veámoslo con un ejemplo.
Podemos consultar el estado de estas reglas con el comando `getsebool -a`.

{% highlight bash %}
[root@salmorejo centos]# getsebool -a | grep httpd
httpd_anon_write --> off
httpd_builtin_scripting --> on
httpd_can_check_spam --> off
httpd_can_connect_ftp --> off
httpd_can_connect_ldap --> off
httpd_can_connect_mythtv --> off
httpd_can_connect_zabbix --> off
httpd_can_network_connect --> on
httpd_can_network_connect_cobbler --> off
httpd_can_network_connect_db --> on
httpd_can_network_memcache --> off
...
{% endhighlight %}

Como podemos comprobar aquí, tenemos habilitadas varias reglas, aunque las dos más importantes son `httpd_can_network_connect --> on` y `httpd_can_network_connect_db --> on` que permiten a las aplicaciones web el acceso al protocolo **http/s** y el acceso a la base de datos.
En el caso de que quisiésemos modificar estas reglas, tendríamos que ejecutar `setsebool -P <nombre política> [0|1]`. Por ejemplo, vamos a habilitar el acceso de las aplicaciones web al **ldap**.

{% highlight bash %}
setsebool -P httpd_can_connect_ldap 1
{% endhighlight %}

Si después consultamos el estado podemos ver esto:

{% highlight bash %}
getsebool httpd_can_connect_ldap
httpd_can_connect_ldap --> on
{% endhighlight %}

## Contextos

Por otro lado tenemos la gestión de los **contextos**, que se realiza mediante el comando **semanage** (que se utilizará para configurar el resto de _SELinux_). Esto nos valdría para cambiar la localización por defecto de determinados servicios, tal y como dijimos antes con el ejemplo de **nginx**.
Entonces el **contexto** no es más que la localización y el uso por defecto de los servicios que instalamos en nuestro sistema. Así como el contexto de **nginx** se localiza en el directorio **/usr/share** y **/etc/nginx**, el contexto de **apache2** se sitúa en **/var/www/** y **/etc/apache/**.
Normalmente tenemos miles de contextos en el sistema, por lo que a la hora de listarlos, solemos filtrarlos con alguna herramienta, por ejemplo con **grep**. Vamos a listar los contextos de acceso vía http a ficheros y/o directorios del sistema:

{% highlight bash %}
[root@salmorejo ~]# semanage fcontext -l | grep httpd_sys_content_t
/etc/htdig(/.*)?                                   all files          system_u:object_r:httpd_sys_content_t:s0 
/srv/([^/]*/)?www(/.*)?                            all files          system_u:object_r:httpd_sys_content_t:s0 
/srv/gallery2(/.*)?                                all files          system_u:object_r:httpd_sys_content_t:s0 
/usr/share/doc/ghc/html(/.*)?                      all files          system_u:object_r:httpd_sys_content_t:s0 
/usr/share/drupal.*                                all files          system_u:object_r:httpd_sys_content_t:s0 
/usr/share/glpi(/.*)?                              all files          system_u:object_r:httpd_sys_content_t:s0 
/usr/share/htdig(/.*)?                             all files          system_u:object_r:httpd_sys_content_t:s0 
/usr/share/icecast(/.*)?                           all files          system_u:object_r:httpd_sys_content_t:s0 
/usr/share/nginx/html(/.*)?                        all files          system_u:object_r:httpd_sys_content_t:s0 
/usr/share/ntop/html(/.*)?                         all files          system_u:object_r:httpd_sys_content_t:s0 
/usr/share/openca/htdocs(/.*)?                     all files          system_u:object_r:httpd_sys_content_t:s0 
/usr/share/selinux-policy[^/]*/html(/.*)?          all files          system_u:object_r:httpd_sys_content_t:s0 
/usr/share/z-push(/.*)?                            all files          system_u:object_r:httpd_sys_content_t:s0 
/var/lib/cacti/rra(/.*)?                           all files          system_u:object_r:httpd_sys_content_t:s0 
/var/lib/htdig(/.*)?                               all files          system_u:object_r:httpd_sys_content_t:s0 
/var/lib/trac(/.*)?                                all files          system_u:object_r:httpd_sys_content_t:s0 
/var/www(/.*)?                                     all files          system_u:object_r:httpd_sys_content_t:s0 
/var/www/icons(/.*)?                               all files          system_u:object_r:httpd_sys_content_t:s0 
/var/www/svn/conf(/.*)?                            all files          system_u:object_r:httpd_sys_content_t:s0
{% endhighlight %}

Observamos que además del contexto por defecto, se han añadido tres lineas más que permiten el acceso en los directorios **/var/www(/.*)?**, **/var/www/icons(/.*)?** y **/var/www/svn/conf(/.*)?**.
Si queremos **añadir** contextos, utilizamos **semanage**, con el parámetro **-a** seguido de **-t** para indicar el tipo de contexto. Si quisiésemos que las apliaciones web de nginx, pudiesen escribir en **/home/usuario/**, tendríamos que aplicar la siguiente regla.

{% highlight bash %}
semanage fcontext -a -t httpd_sys_content_t "/home/usuario(/.*)?"
{% endhighlight %}

Hasta ahora, solo hemos presentado el supuesto en el que las aplicaciones (ejecutadas por el usuario **system**), accedan a ciertos directorios o protocolos, pero esto también se puede aplicar a los usuarios. Si prestamos atención a la salida que obtenemos cuando listamos las **reglas de contexto**, podemos ver en la "tercera columna" varios campos:

{% highlight bash %}
/var/www/svn/conf(/.*)?                            all files          system_u:object_r:httpd_sys_content_t:s0

#########################################
system_u  : object_r : httpd_sys_content_t : s0

usuario_u : rol_r    : tipo_t              : categoría o nivel

{% endhighlight %}

* **usuario_u**: indica el usuario que tendrá acceso al objeto
* **rol**: es el rol del acceso, en este caso un directorio, es decir un objeto.
* **tipo**: nos dice la clase de acceso que estamos concediendo.

## Caso práctico

Una vez explicado a groso modo qué es _SELinux_, cómo funciona y cuales son algunas de sus herramientas de configuración, vamos a pasar a ver un caso práctico donde hemos instalado varios virtualhost **nginx** en un directorio distinto a su contexto por defecto, y utilizando al mismo tiempo servidores de aplicaciones (**php-fpm** y **gunicorn**), además de servirlos por **https**.
Las reglas aplicadas son las siguientes:

{% highlight bash %}
# Permitir acceso de apliaciones web a objetos y a bbdd
setsebool -P httpd_can_network_connect_db 1
setsebool -P httpd_can_network_connect 1

# Permitir el acceso de usuarios ftp a sus respectivos directorios
# también podemos aplicar reglas booleanas con semanage
semanage boolean -m ftpd_full_access --on
semanage boolean -m ftpd_connect_all_unreserved --on

# Nextcloud
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/data(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/config(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/apps(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/.htaccess'
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/.user.ini'
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/3rdparty/aws/aws-sdk-php/src/data/logs(/.*)?'

# Mezzanine
chcon -t httpd_sys_rw_content_t /var/www/mezzanine -R

# SSL/TLS
restorecon -v -R /etc/pki/tls/
{% endhighlight %}

Podemos ver que hemos utilizado varias herramientas, así que vamos por partes. Pero en resumen tenemos dos aplicaciones (aunque un mismo **tipo** de acceso, *httpd_sys_rw_content_t*) y dos nuevas herramientas que antes no hemos visto. Primero vamos con lo conocido.
En las reglas de _nextcloud_ y _mezzanine_, básicamente permitimos al usuario **system** la lectura y escritura por parte de las aplicaciones relacionadas con el protocolo _http_ en esos directorios. Aunque con _nextcloud_ he usado la herramienta **semanage**, con _mezzanine_ he utilizado **chcon**.
Entonces, ¿qué es **chcon** y para qué sirve? Es otra herramienta más, parecida a **semanage** puesto que modifica los contextos, solo que no de forma completamente permanente. Se utiliza en casos donde dicha política es temporal, ya que en el caso de hacer un [relabel](https://www.digrouz.com/mediawiki/index.php/(RHEL)_HOWTO_relabel_a_filesystem_(SELinux)) (como cambiar a otra política y volver a esta) no se conservaría dicha regla.
Por último tenemos la herramienta **restorecon**, que a grandes rasgos, configura automáticamente las reglas necesarias para que se permita al proceso correspondiente acceder al contenido de ese directorio. En mi caso, a los ficheros contenedores de los certificados **ssl**, por parte de **nginx**. Es una herramienta muy útil para aplicarla en momentos donde no tengamos tiempo para pararnos a analizar todos los permisos que necesita la aplicación sobre el objeto.
Cabe mencionar que si en algún momento hubiésemos aplicado alguna regla con **chcon**, **restorecon** la machacaría.
Es por esto que la regla de **Mezzanine** deberíamos cambiarla por una aplicada con **semanage**. En este caso quedaría de la siguiente forma.

{% highlight bash %}
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/mezzanine(/.*)?'
{% endhighlight %}

#### Referencias

* [Relabel SELinux](https://www.digrouz.com/mediawiki/index.php/(RHEL)_HOWTO_relabel_a_filesystem_(SELinux))
* [Diferencia principal entre chcon y semanage](https://superuser.com/questions/669198/semanage-command-not-changing-file-context)
* [Documentación oficial SELinux RHEL](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/index)
* [Modos de funcionamiento en SELinux](https://docs.fedoraproject.org/es-ES/Fedora/13/html/Security-Enhanced_Linux/sect-Security-Enhanced_Linux-Working_with_SELinux-Enabling_and_Disabling_SELinux.html)
