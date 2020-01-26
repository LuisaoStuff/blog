---
title:  "Sistema de ficheros ZFS"
excerpt: "Instalación y configuración de ZFS en Debian Buster"
date:   2020-01-25 10:59:00
categories: [Sistemas]
tags: [zfs,filesystem]
---

## Introducción

**ZFS** es un sistema de ficheros que cuenta con una serie de características avanzadas, tales como _auto-reparación_ o _copy-on-write_. No obstante, aún siendo extremádamente útil en el entorno de servidores, **ZFS** no está tan extendido como cabría esperar. Su licencia **CDDL** (incompatible con la licencia **GPL** de Linux) y diversos acontecimientos durante su desarrollo pusieron bastantes trabas en la implementación de este sistema de ficheros en el _Kernel Linux_.
Dicho esto vamos a simular un escenario con _Vagrant_ donde tendremos una máquina a la que le agregaremos varios discos, para posteriormente, instalar **ZFS** y "jugar" un poco con él.

## Preparación del escenario

Si queréis probarlo, os dejo el [Vagrantfile](/docs/EscenarioZFS/Vagrantfile) donde básicamente creamos una máquina *Debian* con 5 discos adicionales de 400Mb cada uno, y añadimos los repositorios de _back-ports_ de Debian. Estos repositorios nos harán falta a la hora de instalar el paquete, ya que al no tener una licencia **GPL**, el propio _Debian_ no puede incluirlo en la rama _main_ y mucho menos en los paquetes de la instalación.
Una vez creada la máquina, actualizamos el sistema e instalamos el paquete `zfsutils-linux`. Pero antes debemos instalar los `linux-headers`, que en mi caso corresponde con la versión del _kernel_ **4.19.0.6**.
{% highlight bash %}
apt upgrade -y
apt install -y linux-headers-4.19.0-6-amd64
apt install -yt buster-backports dkms spl-dkms
apt install -yt buster-backports zfs-dkms zfsutils-linux
{% endhighlight %}

Para comprobar que **zfs** está funcionando podemos ejecutar:
{% highlight bash %}
root@zfsMachine:~# systemctl status zfs-mount
● zfs-mount.service - Mount ZFS filesystems
   Loaded: loaded (/lib/systemd/system/zfs-mount.service; enabled; vendor preset: enabled)
   Active: active (exited) since Tue 2020-01-21 21:20:39 CET; 19min ago
     Docs: man:zfs(8)
 Main PID: 6631 (code=exited, status=0/SUCCESS)
    Tasks: 0 (limit: 1150)
   Memory: 0B
   CGroup: /system.slice/zfs-mount.service

ene 21 21:20:39 zfsMachine systemd[1]: Starting Mount ZFS filesystems...
ene 21 21:20:39 zfsMachine systemd[1]: Started Mount ZFS filesystems.
{% endhighlight %}

## Creación y configuración de los "pool"

Un _pool_ en **ZFS** sería algo así lo que un grupo de volúmenes en **LVM**. La principal diferencia es que con **ZFS** fusionamos lo que es la capa de volúmenes lógicos y la capa de _RAID_ software. Es por esto que durante la creación y configuración de un _pool_, podemos definir el tipo de _RAID_ que vamos a crear, así como otras diversas opciones; discos caché, tamaño de sectores, etc.
Los tipos de _RAID_ soportados por **ZFS** son los siguientes.

* [RAID0](https://es.wikipedia.org/wiki/RAID#RAID_0_(Data_Striping,_Striped_Volume)). Es igual que el tradicional. Mejora velocidad lectura/escritura pero _no ofrece redundancia_.
* [RAID1](https://es.wikipedia.org/wiki/RAID#RAID_1_(espejo)). Es igual que el tradicional. Ofrece una redundancia igual a **n-1**
* [RAID10](https://es.wikipedia.org/wiki/RAID#RAID_1+0). No hay diferencia con el tradicional. Es la combinación del _RAID_ 1 y 0. Es decir, es la agrupación de _RAID 1_ en _RAID 0_.
* [RAIDZ](https://es.wikipedia.org/wiki/RAID#RAID_Z) **1**, **2** y **3**. Dependiendo del tipo tiene uno, dos o tres bit de paridad, necesitándose 3, 4 o 5 discos respectivamente.

En esta ocasión vamos a trabar con **RAIDZ-2** por lo que necesitaremos **4** discos. La creación de estos _pool_ se realiza con el comando `zpool`.

{% highlight bash %}
zpool create -f EjemploRaidZ raidz2 /dev/sdb /dev/sdc /dev/sdd /dev/sde
zpool status
  pool: EjemploRaidZ
 state: ONLINE
  scan: none requested
config:

	NAME          STATE     READ WRITE CKSUM
	EjemploRaidZ  ONLINE       0     0     0
	  raidz2-0    ONLINE       0     0     0
	    sdb       ONLINE       0     0     0
	    sdc       ONLINE       0     0     0
	    sdd       ONLINE       0     0     0
	    sde       ONLINE       0     0     0

errors: No known data errors
{% endhighlight %}

De la misma forma que en los _RAID_ tradicionales (como cuando los gestionamos con `mdadm`), podemos añadir discos de reserva (_hot spare_). De esta manera, si uno de los discos falla, el _hot spare_ se reemplazaría de forma automática. Para añadir un disco en modo reserva ejecutamos:

{% highlight bash %}
zpool add EjemploRaidZ spare sdf
zpool status
  pool: EjemploRaidZ
 state: ONLINE
  scan: none requested
config:

	NAME          STATE     READ WRITE CKSUM
	EjemploRaidZ  ONLINE       0     0     0
	  raidz2-0    ONLINE       0     0     0
	    sdb       ONLINE       0     0     0
	    sdc       ONLINE       0     0     0
	    sdd       ONLINE       0     0     0
	    sde       ONLINE       0     0     0
	spares
	  sdf         AVAIL

errors: No known data errors
{% endhighlight %}

## Simulación de fallos y pruebas de redundancia

Vamos a ejecutar una serie de comandos para simular un fallo de un disco con el parámetro `offline` y vamos a obsevar como se reemplaza automáticamente y sin perder información. Antes vamos a crear varios ficheros para comprobar que siguen manteniendo su integridad. 
Aunque a priori la gestión de **zfs** pueda parecerse en gran medida a la gestión de _RAID_ con **mdadm**, la verdad es que conceptualmente no tienen nada que ver. Ya de primeras cuando gestionamos un _RAID_ software con **mdadm**, necesitamos (tras haber particionado, aplicado una capa de _LVM_, o no) montar el dispositivo de bloques que se nos ha generado. En contraposición, el _pool_ que se nos genera con **zfs** se monta automáticamete en la raiz del sistema con el nombre que le hayamos asignado. No obstante, también podríamos generar volúmenes y posteriormente darles el formato que queramos (con la ventaja de que seguiría funcionando por debajo con **zfs**). Aunque esto último lo veremos en el siguiente apartado.
Por el momento vamos a generar una serie de ficheros en **/EjemploRaidZ**.

{% highlight bash %}
#Creamos un arbol de directorios
mkdir -p /EjemploRaidZ/directorio/subdirectorio/
#Creamos ficheros aleatorios
dd if=/dev/urandom of=/EjemploRaidZ/ficheroRandom bs=64K count=300
dd if=/dev/urandom of=/EjemploRaidZ/directorio/ficheroRandom bs=64K count=600
echo "Prueba de un fichero" > /EjemploRaidZ/directorio/subdirectorio/fichtest
{% endhighlight %}

Antes de hacer fallar uno de los discos, vamos a generar un _checksum_ de los dos ficheros aleatorios con el comando `md5sum`. De esta manera si cambia aunque sea un _byte_ de alguno de los dos ficheros, el _checksum_ será completamente distinto.

{% highlight bash %}
md5sum ficheroRandom 
935a8851f013f41e0dab7afad20b7377  ficheroRandom
md5sum directorio/ficheroRandom 
49b5591a6b7daecbecd185bae9e6f769  directorio/ficheroRandom
{% endhighlight %}

Para simular el fallo del disco, vamos a ejecutar el siguiente comando.

{% highlight bash %}
zpool offline -f EjemploRaidZ sdc
zpool status
  pool: EjemploRaidZ
 state: DEGRADED
status: One or more devices are faulted in response to persistent errors.
	Sufficient replicas exist for the pool to continue functioning in a
	degraded state.
action: Replace the faulted device, or use 'zpool clear' to mark the device
	repaired.
  scan: none requested
config:

	NAME          STATE     READ WRITE CKSUM
	EjemploRaidZ  DEGRADED     0     0     0
	  raidz2-0    DEGRADED     0     0     0
	    sdb       ONLINE       0     0     0
	    sdc       FAULTED      0     0     0  external device fault
	    sdd       ONLINE       0     0     0
	    sde       ONLINE       0     0     0
	spares
	  sdf         AVAIL

errors: No known data errors

md5sum ficheroRandom 
935a8851f013f41e0dab7afad20b7377  ficheroRandom
md5sum directorio/ficheroRandom 
49b5591a6b7daecbecd185bae9e6f769  directorio/ficheroRandom
{% endhighlight %}

Como podemos comprobar, aunque el _RAID_ esté degradado, gracias a que tiene una tolerancia a fallos (gracias a los 2 _bits de paridad_ repartidos por todos los discos), seguimos obteniendo los mismos _checksum_ por lo tanto la **integridad** de los ficheros está **garantizada**. No obstante ahora mismo nos encontramos sin redundancia, por lo que si quisiésemos acoplar el disco de reserva solo tenemos que ejecutar:

{% highlight bash %}
zpool replace -f EjemploRaidZ sdc sdf
zpool status
  pool: EjemploRaidZ
 state: DEGRADED
status: One or more devices are faulted in response to persistent errors.
	Sufficient replicas exist for the pool to continue functioning in a
	degraded state.
action: Replace the faulted device, or use 'zpool clear' to mark the device
	repaired.
  scan: resilvered 28.7M in 0 days 00:00:00 with 0 errors on Sun Jan 26 17:27:07 2020
config:

	NAME          STATE     READ WRITE CKSUM
	EjemploRaidZ  DEGRADED     0     0     0
	  raidz2-0    DEGRADED     0     0     0
	    sdb       ONLINE       0     0     0
	    spare-1   DEGRADED     0     0     0
	      sdc     FAULTED      0     0     0  external device fault
	      sdf     ONLINE       0     0     0
	    sdd       ONLINE       0     0     0
	    sde       ONLINE       0     0     0
	spares
	  sdf         INUSE     currently in use

errors: No known data errors
{% endhighlight %}

Ahora tenemos dos opciones; podemos indicar que hemos reparado el disco que nos ha fallado, o podemos eliminarlo del _RAID_ y dejar funcionando el disco que antes teníamos de reserva. Si optamos por la primera opción, tal y como nos indica la propia salida del estado del _zpool_, ejecutamos `zpool clear EjemploRaidZ sdc` para indicar que ya hemos reparado el disco.
Si por el contrario, el disco que nos ha fallado no podemos repararlo, el método correcto sería eliminarlo del _RAID_ (dejar trabajando al antiguo disco de reserva) y añadir un nuevo disco como reserva. En este segundo escenario ejecutaríamos `zpool detach EjemploRaidZ sdc` para extraerlo del _RAID_ y después añadir otro disco como reserva, tal y como hicimos antes.

### Restauración de un RAID con "rollback"

**ZFS** tiene una funcionalidad parecida al _rollback_ de las bases de datos de _Oracle_, y es que en un determinado momento en el que hemos tenido un número de fallos mayor al nivel de redundancia que teníamos disponible, podemos volver a un estado de este mismo _RAID_, anteior a dichos fallos.
Para poder hacer esto primero tendremos que guardar una _snapshot_, que será nuestro "punto de guardado" al que podremos volver posteriormente. Para crearlo tan solo ejecutamos:

{% highlight bash %}
zpool checkpoint EjemploRaidZ
{% endhighlight %}

Vamos a generar los _checksum_ igual que antes para comprobar la integridad de los archivos una vez que restauremos el **checkpoint**.

{% highlight bash %}
md5sum directorio/ficheroRandom 
49b5591a6b7daecbecd185bae9e6f769  directorio/ficheroRandom
md5sum ficheroRandom 
935a8851f013f41e0dab7afad20b7377  ficheroRandom
{% endhighlight %}

Ahora establecemos como **FAULTED** dos de los discos del _RAID_.

{% highlight bash %}
zpool offline -f EjemploRaidZ sdc
zpool offline -f EjemploRaidZ sde
zpool status
  pool: EjemploRaidZ
 state: DEGRADED
status: One or more devices are faulted in response to persistent errors.
	Sufficient replicas exist for the pool to continue functioning in a
	degraded state.
action: Replace the faulted device, or use 'zpool clear' to mark the device
	repaired.
  scan: resilvered 28.6M in 0 days 00:00:00 with 0 errors on Sun Jan 26 17:37:59 2020
checkpoint: created Sun Jan 26 17:49:02 2020, consumes 482K
config:

	NAME          STATE     READ WRITE CKSUM
	EjemploRaidZ  DEGRADED     0     0     0
	  raidz2-0    DEGRADED     0     0     0
	    sdb       ONLINE       0     0     0
	    sdf       FAULTED      0     0     0  external device fault
	    sdd       ONLINE       0     0     0
	    sde       FAULTED      0     0     0  external device fault

errors: No known data errors

{% endhighlight %}

Para restaurar el estado del _RAID_, lo exportamos e importamos utilizando el parámtero `--rewind-to-checkpoint`.

{% highlight bash %}
zpool export EjemploRaidZ
zpool import --rewind-to-checkpoint EjemploRaidZ
zpool status
  pool: EjemploRaidZ
 state: ONLINE
  scan: none requested
config:

	NAME          STATE     READ WRITE CKSUM
	EjemploRaidZ  ONLINE       0     0     0
	  raidz2-0    ONLINE       0     0     0
	    sdb       ONLINE       0     0     0
	    sdf       ONLINE       0     0     0
	    sdd       ONLINE       0     0     0
	    sde       ONLINE       0     0     0
{% endhighlight %}

Y comprobamos los _checksum_

{% highlight bash %}
md5sum ficheroRandom 
935a8851f013f41e0dab7afad20b7377  ficheroRandom
md5sum directorio/ficheroRandom 
49b5591a6b7daecbecd185bae9e6f769  directorio/ficheroRandom
{% endhighlight %}

Como podemos ver, todo está correcto!

### Copy on write [COW]

Básicamente **COW** es una funcionalidad que previene la corrupción de los datos en caso de un apagado inesperado del sistema durante la modificación de ficheros. Cuando modificamos un fichero en _ZFS_, realmente estamos modificando una copia que apunta al fichero orginal, por lo que si se detiene el proceso de modificación de forma abrupta, el fichero original no sufrirá daños. Para probar esto vamos a modificar un fichero con **nano** y sin salir del editor vamos a forzar un apagado con **Virtualbox**.
Si todo sale como es debido, el fichero debería conservarse con la linea que guardamos antes "_hola esto es una prueba_".

<a href="/images/cow-test.png"><img src="/images/cow-test.png" /></a>

Comprobamos después de iniciar la máquina que el fichero ha conservado la integridad.

{% highlight bash %}
cat /EjemploRaidZ/testCOW
hola esto es una prueba
{% endhighlight %}

### Creación de volúmenes lógicos

Esta funcionalidad es de gran utilidad ya que podríamos utilizar _ZFS_ como base de una [SAN](https://es.wikipedia.org/wiki/Red_de_%C3%A1rea_de_almacenamiento), gracias a que estos volúmenes se comportan como cualquier dispositivo de bloques a ojos del sistema operativo. Podríamos particionarlos, darles un formato, crear un volúmenes lógicos con _LVM_, y todo esto funcionando por debajo _ZFS_, incluyendo todas las ventajas que hemos mencionado antes!
Para crear estos dispositivos de bloques necesitamos tener un _zpool_, como el que hicimos antes. La sintaxis básica sería la siguiente:

{% highlight bash %}
zfs create -V [tamaño] [nombre-del-pool]/[nombre-del-volumen]
{% endhighlight %}

En mi caso sería:

{% highlight bash %}
zfs create -V 200mb EjemploRaidZ/vol1
zfs list
NAME                USED  AVAIL     REFER  MOUNTPOINT
EjemploRaidZ        209M   428M     35.9K  /EjemploRaidZ
EjemploRaidZ/vol1   208M   637M     17.9K  -
{% endhighlight %}

Para poder darle algún tipo de uso dentro del sistema, tendríamos que darle un formato reconocible, en este caso lo formatearemos con **ext4** y lo montaremos en **/home/vagrant/vol1**:

{% highlight bash %}
mkfs.ext4 /dev/zvol/EjemploRaidZ/vol1
mount /dev/zvol/EjemploRaidZ/vol1 /home/vagrant/vol1
{% endhighlight %}

Comprobamos como se ha montado

{% highlight bash %}
lsblk -f
NAME   FSTYPE     LABEL        UUID                                 FSAVAIL FSUSE% MOUNTPOINT

...
zd0                                                                  174.2M     1% /home/vagrant/vol1
{% endhighlight %}

### Snapshots

El sistema de _snapshots_ utilizado por **ZFS** permite realizar instantáneas del sistema que, se almacenan en el mismo dispositivo de bloques donde se está utilizando dicho sistema de ficheros. Esto nos impediría utilizarlas en caso de desastre, pero siempre podemos recurrir a la replicación y copiarlas en otro dispositivo separado de **ZFS**.
Aunque al principio estas _snapshots_ no ocupan espacio, comenzarán a crecer en función de los cambios que experimente el sistema desde que se creó la instantánea.
Para crear o eliminar un _snapshot_ la sintaxis es la siguiente.
{% highlight bash %}
zfs snapshot zpool/vol@nombre-de-snapshot
zfs destroy zpool/vol@nombre-de-snapshot
{% endhighlight %}

Y en caso de que quisiésemos volver a una _snapshot_ que ya hicimos con anterioridad ejecutaríamos un comando como este:

{% highlight bash %}
# utilizaríamos el parámetro -r en caso de que
# quisiésemos eliminar automáticamente la snapshot
# tras hacer el "rollback"
zfs rollback -r zpool/vol@nombre-de-snapshot
{% endhighlight %}

### Otras funcionalidades

Tal y como hemos visto a lo largo de esta entrada, **ZFS** es verdaderamente un sistema de ficheros avanzado, pero no se queda ahí. Además de todo lo que ya hemos visto (COW, RAID, Volumenes, etc), dispone de otras aplicaciones.

#### Discos caché y log en RAID

Para un aumento del rendimient _ZFS_ nos ofrece la posibilidad de añadir discos caché. Esto tendría sentido en el supuesto de tener el almacenamiento principal del _RAID_ en discos mecánicos y añadir un disco de estado sólido como un [M.2 NVME PCIE](https://youtu.be/ANVvxlv6DV4).
También nos permite añadir especificar un disco **log**. Inicialmente pensaríamos que es un disco para guardar los ficheros de _log_ del sistema, como los de las unidades _systemd_, aplicaciones como _apache2_, _nginx_, etc. Sin embargo, es una unidad también de tipo caché, que se encarga de hacer más liviano el _copy-on-write_, sobre todo cuando se necesitan estructuras síncronas. A este tipo de dispositivo se le conoce como **ZFS Intent Log** o **ZIL**.
Si quisiésemos definir este tipo de unidades en el _RAID_, tan solo tenemos que añadir a la derecha de la unidad en cuestión el parámetro `cache` en caso de disco caché convencional o `log` en caso de un disco orientado al **ZIL** que hemos explicado antes.


## Conclusión

**ZFS** es verdaderamente un sistema de ficheros avanzado. Aporta una serie de funcionalidades que lo hacen sumamente útil en el caso de que necesitemos un sistema de ficheros _robusto_. Más allá de la _redundancia_ que aporta según el tipo de _RAID_ que utilicemos, hemos podido comprobar otras funciones adicionales respecto a los sistemas tradicionales como el _copy-on-write_ o la posibilidad de generar dispositivos de bloques completamente funcionales, además de la creación de _snapshots_.
No obstante, debido a la complejidad de su instalación y la incompatibilidad de la licencia **CDDL**, incompatible con la licencia de _Linux_ **GPL**, hace que su uso no esté tan extendido.
Dicho esto, aunque creo que el uso **ZFS** merece la pena debido a todo el potencial que tiene, su licencia lo condenó a caer cada vez más en el desuso, y todavía más con el desarrollo contínuo de **BTRFS**, el cual sí tiene una licencia compatible con el _Kernel Linux_
