---
title:  "Despliegue de Kerberos con OpenLDAP y NFS4"
excerpt: "Instalación básica del sistema de autenticación centralizado Kerberos con OpenLDAP y gestión de los ficheros de usuario con NFS4"
date:   2020-03-04 10:00:00
categories: [Sistemas, Seguridad]
---
## Introdicción


## 

Aprovechamos la instalación de bind9 y openldap.

Añadimos registro de ldap al dns.

Instalamos en tortilla el cliente ldap, pero sin instalar el sistema de autenticación por defecto:

apt install --no-install-recommends libnss-ldap


Nos aparecerá una consola de configuración.

ldap://ldap.luis.gonzalonazareno.org

Identificador del servidor LDAP: dc=luis,dc=gonzalonazareno,dc=org

Versión de LDAP: 3


Nos dirigimos al fichero /etc/nsswitch.conf y modificamos los siguientes campos para que queden así:

{% highlight bash %}
passwd:         ldap files
group:          ldap files
{% ehighlight %}



creamos un usuario nuevo:


dn: uid=luis,ou=People,dc=luis,dc=gonzalonazareno,dc=org
objectClass: top
objectClass: posixAccount
objectClass: inetOrgPerson
cn:: Luis Vazquez
sn: vazquez
uid: luis
uidNumber: 3020
gidNumber: 3020
homeDirectory: /home/luis
loginShell: /bin/bash
userPassword: {SSHA}RXfwOxB/24AUXi8lVDunxclMeMBC+QVf
mail: luisvazquezalejo@gmail.com
givenName: luis



















Cambiar contraseña admin LDAP.

generamos una contraseña con `slappasswd`

{% highlight bash %}
slappasswd
New password: 
Re-enter new password: 
{SSHA}GWFGyj31w7aDpgmQXsl290PolQLI3KK5
{% endhighlight %}


ejecutamos el siguiente comando

{% highlight bash %}
ldapmodify -Y EXTERNAL -H ldapi:///
{% endhighlight %}

e introducimos los siguientes campos

{% highlight bash %}
dn: olcDatabase={1}mdb,cn=config
replace: olcRootPW
olcRootPW: {SSHA}GWFGyj31w7aDpgmQXsl290PolQLI3KK5
{% endhighlight %}

Volvemos a introducir un retorno de carro (pulsamos _enter_)

{% highlight bash %}
modifying entry "olcDatabase={1}mdb,cn=config"
{% endhighlight %}

Después de esto podemos introducir un **^C** para salir y luego reiniciaríamos el servicio de **slapd**.

