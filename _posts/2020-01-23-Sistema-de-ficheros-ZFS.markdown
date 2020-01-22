---
title:  "Sistema de ficheros ZFS"
excerpt: "Instalación y configuración de ZFS"
date:   2020-01-23 10:59:00
categories: [Sistemas]
tags: [zfs,filesystem]
---

## Introducción

**ZFS** es un sistema de ficheros que cuenta con una serie de características avanzadas, tales como _auto-reparación_ o _copy-on-write_. No obstante, aún siendo extremádamente útil en el entorno de servidores, **ZFS** no está tan extendido como cabría esperar. Su licencia **CDDL** (incompatible con la licencia **GPL** de Linux) y diversos acontecimientos durante su desarrollo pusieron bastantes trabas en la implementación de este sistema de ficheros en el _Kernel Linux_.
Dicho esto vamos a simular un escenario con _Vagrant_ donde tendremos una máquina a la que le agregaremos varios discos, para posteriormente, instalar **ZFS** y "jugar" un poco con él.

## Preparación del escenario

Si queréis probarlo, os dejo el [Vagrantfile](/docs/EscenarioZFS/Vagrantfile) donde básicamente creamos una máquina *Debian* con 5 discos adicionales de 400Mb cada uno, y añadimos los repositorios de _back-ports_ de Debian. Estos repositorios nos harán falta a la hora de instalar el paquete, ya que al no tener una licencia **GPL**, el propio _Debian_ no puede incluirlo en la rama _main_ y mucho menos en los paquetes de la instalación.
Una vez creada la máquina, actualizamos el sistema e instalamos el paquete `zfsutils-linux`.
{% highlight bash %}
apt install zfsutils-linux
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

## 









CONFIGURACIÓN DISCOS EN RAID

Creamos un raid 1:

root@debian:~# zpool create -f Raid1 mirror /dev/sdb /dev/sdc /dev/sde

Comprobamos que se ha creado correctamente:

root@debian:~# zpool status
  pool: Raid1
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	Raid1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0
	    sde     ONLINE       0     0     0

errors: No known data errors

PRUEBAS DE FALLO Y SUSTITUCIÓN

Ahora añadimos un disco en modo "spare":

root@debian:~# zpool add -f Raid1 spare sdd

Hacemos que el dispositivo sde nos de un error:

root@debian:~# zpool offline -f Raid1 sde
root@debian:~# zpool status
  pool: Raid1
 state: DEGRADED
status: One or more devices are faulted in response to persistent errors.
	Sufficient replicas exist for the pool to continue functioning in a
	degraded state.
action: Replace the faulted device, or use 'zpool clear' to mark the device
	repaired.
  scan: resilvered 304K in 0 days 00:00:00 with 0 errors on Sun Jan 19 18:26:33 2020
config:

	NAME        STATE     READ WRITE CKSUM
	Raid1       DEGRADED     0     0     0
	  mirror-0  DEGRADED     0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0
	    sde     FAULTED      0     0     0  external device fault
	spares
	  sdd       AVAIL

errors: No known data errors

Ahora reemplazamos la unidad que nos da el fallo por la que pusimos en modo "spare":

root@debian:~# zpool replace -f Raid1 /dev/sde /dev/sdd
root@debian:~# zpool status
  pool: Raid1
 state: DEGRADED
status: One or more devices are faulted in response to persistent errors.
	Sufficient replicas exist for the pool to continue functioning in a
	degraded state.
action: Replace the faulted device, or use 'zpool clear' to mark the device
	repaired.
  scan: resilvered 363K in 0 days 00:00:00 with 0 errors on Sun Jan 19 18:51:32 2020
config:

	NAME         STATE     READ WRITE CKSUM
	Raid1        DEGRADED     0     0     0
	  mirror-0   DEGRADED     0     0     0
	    sdb      ONLINE       0     0     0
	    sdc      ONLINE       0     0     0
	    spare-2  DEGRADED     0     0     0
	      sde    FAULTED      0     0     0  external device fault
	      sdd    ONLINE       0     0     0
	spares
	  sdd        INUSE     currently in use

errors: No known data errors

Una vez hecho esto, marcamos como reparado el que tiene el fallo, y vemos como el que estaba antes en modo "spare" vuelve a su lugar automáticamente (como hemos hecho el status rápidamente, hemos podido ver como el sde se estaba "reparando"):

root@debian:~# zpool clear Raid1 sde
root@debian:~# zpool status
  pool: Raid1
 state: ONLINE
status: One or more devices is currently being resilvered.  The pool will
	continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Sun Jan 19 18:52:55 2020
	21K scanned at 21K/s, 21K issued at 21K/s, 186K total
	63K resilvered, 11,29% done, no estimated completion time
config:

	NAME         STATE     READ WRITE CKSUM
	Raid1        ONLINE       0     0     0
	  mirror-0   ONLINE       0     0     0
	    sdb      ONLINE       0     0     0
	    sdc      ONLINE       0     0     0
	    spare-2  ONLINE       0     0     0
	      sde    ONLINE       0     0     0  (resilvering)
	      sdd    ONLINE       0     0     0
	spares
	  sdd        INUSE     currently in use

errors: No known data errors

root@debian:~# zpool status
  pool: Raid1
 state: ONLINE
  scan: resilvered 63K in 0 days 00:00:00 with 0 errors on Sun Jan 19 18:52:55 2020
config:

	NAME        STATE     READ WRITE CKSUM
	Raid1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0
	    sde     ONLINE       0     0     0
	spares
	  sdd       AVAIL   

errors: No known data errors

RESTAURACIÓN RAID

Primero, comprobamos que todo el raid esta online y guardamos la situación actual del raid mediante el siguiente comando:

root@debian:~# zpool status
  pool: Raid1
 state: ONLINE
  scan: resilvered 606K in 0 days 00:00:01 with 0 errors on Sun Jan 19 19:44:16 2020
config:

	NAME        STATE     READ WRITE CKSUM
	Raid1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0
	    sde     ONLINE       0     0     0
	spares
	  sdd       AVAIL   

errors: No known data errors
root@debian:~# zpool checkpoint Raid1

Tras esto, ponemos un fallo en un par de discos:

root@debian:~# zpool offline -f Raid1 sde
root@debian:~# zpool offline -f Raid1 sdc
root@debian:~# zpool status
  pool: Raid1
 state: DEGRADED
status: One or more devices are faulted in response to persistent errors.
	Sufficient replicas exist for the pool to continue functioning in a
	degraded state.
action: Replace the faulted device, or use 'zpool clear' to mark the device
	repaired.
  scan: none requested
checkpoint: created Sun Jan 19 20:00:54 2020, consumes 82,5K
config:

	NAME        STATE     READ WRITE CKSUM
	Raid1       DEGRADED     0     0     0
	  mirror-0  DEGRADED     0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     FAULTED      0     0     0  external device fault
	    sde     FAULTED      0     0     0  external device fault
	spares
	  sdd       AVAIL   

errors: No known data errors

Finalmente, para volver al que teníamos guardado con el checkpoint, hacemos lo siguiente y comprobamos:

root@debian:~# zpool export Raid1
root@debian:~# zpool import --rewind-to-checkpoint Raid1
root@debian:~# zpool status
  pool: Raid1
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	Raid1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0
	    sde     ONLINE       0     0     0
	spares
	  sdd       AVAIL   

errors: No known data errors

VENTAJAS E INCONVENIENTES RESPECTO A MDADM

- mdadm es GPL por lo que esta incluida en el kernel Linux, mientras que ZFS no.
- A diferencia de mdadm, ZFS se apoya en espacios de almacenamiento virtuales (storage pools), los cuales se construyen a partir de uno o más dispositivos virtuales.
- En mdadm, cuando los datos son sobreescritos, se pierden para siempre; mientras que en ZFS, la nueva información se escribe en bloques diferentes, y los datos modificados se escriben en él. Esto asegura que en caso de que ocurra algún tipo de fallo en la sobreescritura, los datos antiguos serán preservados.
- En ZFS, una snapshot se usa principalmente para hacer un seguimiento de los cambios de uno o varios ficheros, pero no de la creación de los mismos. Con esto es posible volver a versiones anteriores de ficheros que nos interesen; aunque hay que tener en cuenta que si un fichero es borrado, el snapshot se borra con él.
- Tamaño máximo: 256 cuatrillones de Zettabytes || Tamaño máximo de archivo: 16 Exabytes

5.

COMPRESIÓN

Con el siguiente comando habilitamos la compresión:

root@debian:~# zfs set compression=on Raid1

Comprobamos:

root@debian:~# zfs get compression Raid1
NAME   PROPERTY     VALUE     SOURCE
Raid1  compression  on        local

Además podemos ver el algoritmo de compresión usado, que en este caso vemos que es lz4:

root@debian:~# zpool get all Raid1
...
Raid1  feature@lz4_compress           active                         local
...

Ahora hacemos una prueba real, viendo el ratio de compresión antes de empezar:

root@debian:~# zfs get compressratio Raid1
NAME   PROPERTY       VALUE  SOURCE
Raid1  compressratio  1.00x  -

Ahora agrupamos por ejemplo 2 directorios en un archivo .tar:

root@debian:~# sudo tar -cf /Raid1/fernando.tar /etc/ /var/log/
tar: Eliminando la `/' inicial de los nombres
tar: Eliminando la `/' inicial de los objetivos de los enlaces
root@debian:~# ls -lh /Raid1/
enc/          fernando.tar  

Vemos el tamaño del archivo:

root@debian:~# ls -lh /Raid1/fernando.tar 
-rw-r--r-- 1 root root 19M ene 20 19:25 /Raid1/fernando.tar

Podemos ver que el ratio de compresión ha aumentado:

root@debian:~# zfs get compressratio Raid1
NAME   PROPERTY       VALUE  SOURCE
Raid1  compressratio  2.26x  -

DEDUPLICACIÓN

La deduplicación es una técnica de compresión para eliminar copias duplicadas de datos repetidos. Esto optimiza el almacenamiento y reduce la información que debe enviarse de un dispositivo a otro a través de redes de comunicación.

Por defecto, la deduplicación esta desactivada:

root@debian:~# zfs get dedup Raid1
NAME   PROPERTY  VALUE          SOURCE
Raid1  dedup     off            default

Con el siguiente comando la activamos:

root@debian:~# zfs set dedup=on Raid1

Comprobamos:

root@debian:~# zfs get dedup Raid1
NAME   PROPERTY  VALUE          SOURCE
Raid1  dedup     on             local

Ahora hacemos una prueba real, primero vemos el ratio de duplicación:

root@debian:/Raid1# zpool get dedupratio Raid1
NAME   PROPERTY    VALUE  SOURCE
Raid1  dedupratio  1.00x  -

Tras esto, vamos a duplicar el fichero que realizamos anteriormente:

root@debian:/Raid1# sudo cp /Raid1/fernando.tar{,.2}

Vemos que el fichero que se ha generado ocupa menos espacio:

root@debian:/Raid1# ls -sh
total 15M
 512 enc  8,1M fernando.tar  6,4M fernando.tar.2

Y que el ratio de duplicación ha subido:

root@debian:/Raid1# zpool get dedupratio Raid1 
NAME   PROPERTY    VALUE  SOURCE
Raid1  dedupratio  2.00x  -
