---
title:  "Introducción a ISCSI"
excerpt: "Instalación y configuración de un servidor y clientes ISCSI"
date:   2020-02-03 10:59:00
categories: [Sistemas]
tags: [ISCSI,filesystem]
---

## Introducción

**ISCSI** es un estandar del protocolo **SCSI** que permite la distribución de dispositivos de bloques a través de **TCP/IP**. Dicho protocolo se utiliza por norma general en escenarios donde se ve involucrada una [SAN](https://es.wikipedia.org/wiki/Red_de_%C3%A1rea_de_almacenamiento) o al menos una [NAS](https://es.wikipedia.org/wiki/Almacenamiento_conectado_en_red), que se encargan de proveer almacenamiento en la red.
Gracias a este estandar podemos tener centralizado el almacenamiento de múltiples máquinas y hasta podríamos hacer que estas arrancasen con un dispositivo de bloques proporcionado por **ISCSI**.
Para ver su funcionamiento, vamos a plantear un escenario donde tendremos un servidor (_debian_) y dos clientes (_debian_ y _windows_). Si queréis replicar el escenario, os dejo el [Vagrantfile](/docs/EscenarioISCSI/Vagrantfile) que he utilizado. Intentaremos conseguir los siguientes objetivos:

* Crea un target con una LUN y conéctala a un cliente GNU/Linux. Explica cómo escaneas desde el cliente buscando los targets disponibles y utiliza la unidad lógica proporcionada, formateándola si es necesario y montándola.
* Utiliza systemd mount para que el target se monte automáticamente al arrancar el cliente.
* Crea un target con 2 LUN y autenticación por CHAP y conéctala a un cliente windows. Explica cómo se escanea la red en windows y cómo se utilizan las unidades nuevas (formateándolas con NTFS).

_Para más información sobre ISCSI consulta_ [aquí](https://es.wikipedia.org/wiki/ISCSI).



## Configuración del servidor

Antes de configurar **ISCSI** en el servidor, vamos a crear un _RAIDZ-2_, tal y como hicimos en la entrada de [ZFS](/Sistema-de-ficheros-ZFS/). Crearemos el _raid_, nombrandolo como **ISCSI**, formado por 4 de los discos que hemos añadido. Después añadiremos un disco de reserva y crearemos 3 volúmenes lógicos, por lo que seguimos estas instrucciones.

{% highlight bash %}
zpool create -f ISCSI raidz2 /dev/sdb /dev/sdc /dev/sdd /dev/sde spare /dev/sfd
zfs create -V 500mb ISCSI/vol1
zfs create -V 500mb ISCSI/vol2
zfs create -V 500mb ISCSI/vol3
{% endhighlight %}

Reitero que esto _no es necesario_. Si queréis replicar el ejercicio, solo **necesitáis** terminar con una serie de **volúmenes lógicos**, o **dispositivos de bloques** reconocibles por el sistema, es decir, que podáis encontrarlos en **/dev/**.
Dicho esto, pasamos a la configuración del servidor **ISCSI**. En este caso vamos a utilizar el paquete **tgt**, por lo que lo instalamos con **apt**.

{% highlight bash %}
apt install tgt
{% endhighlight %}

## Caso 1. Target con 1 LUN y cliente Linux anónimo

### Configuración del servidor

Y procedemos a crear el primer **target** con el siguiente comando.

{% highlight bash %}
tgtadm --lld iscsi --op new --mode target --tid 1 -T iqn.2020-02.es.luisvazquezalejo:target1
{% endhighlight %}

De aquí nos quedamos sobre todo con la sintáxis del último campo que determina la dirección y el propio nombre del **target**. Es decir la sintaxis sería

{% highlight bash %}
iqn.año-mes.es.dominio:nombre-del-target
{% endhighlight %}

Después añadimos la unidad lógica al target.

{% highlight bash %}
tgtadm --lld iscsi --op new --mode logicalunit --tid 1 --lun 1 -b /dev/ISCSI/vol1
{% endhighlight %}

Podemos ver el estado del **target** ejecutando `tgtadm --lld iscsi --op show --mode target`, obteniendo una salida como [esta](https://cutt.ly/9rIIQZB). Por último solo tenemos que introducir el siguiente comando para crear la _ACL_ correspondiente y de esta forma que los clientes puedan encontrar la _LUN_.

{% highlight bash %}
tgtadm --lld iscsi --op bind --mode target --tid 1 -I ALL
{% endhighlight %}

### Configuración del cliente [ISCSI Initiator]

La configuración de un cliente _Linux_ para que use los _target_ de ISCSI, es bastante simple. Para empezar tenemos que instalar **open-iscsi**, por lo que ejecutamos.

{% highlight bash %}
apt install open-iscsi
{% endhighlight %}

Después nos dirigimos al fichero **/etc/iscsi/initiatorname.iscsi** y modificamos última linea, estableciendo la dirección utilizará más tarde el demonio de _iscsi_. En mi caso queda de la siguiente forma.

{% highlight bash %}
## DO NOT EDIT OR REMOVE THIS FILE!
## If you remove this file, the iSCSI daemon will not start.
## If you change the InitiatorName, existing access control lists
## may reject this initiator.  The InitiatorName must be unique
## for each iSCSI initiator.  Do NOT duplicate iSCSI InitiatorNames.
InitiatorName=iqn.2020-02.es.luisvazquezalejo:target1
{% endhighlight %}

De momento, esto no lo vamos a necesitar, ya que primero vamos a buscar e iniciar el _target_ manualmente. Para hacer esto tenemos que introducir las siguientes instrucciones.

{% highlight bash %}
# Escaneamos los targets de la máquina 192.168.100.10
iscsiadm -m discovery -t st -p 192.168.100.10
192.168.100.10:3260,1 iqn.2020-02.es.luisvazquezalejo:target1
# Iniciamos el target
root@clienteLinux:/home/vagrant# iscsiadm -m node -T iqn.2020-02.es.luisvazquezalejo:target1 -p 192.168.100.10 -l
Logging in to [iface: default, target: iqn.2020-02.es.luisvazquezalejo:target1, portal: 192.168.100.10,3260] (multiple)
Login to [iface: default, target: iqn.2020-02.es.luisvazquezalejo:target1, portal: 192.168.100.10,3260] successful.
# Comprobamos que se ha iniciado
lsblk -f
NAME FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
sda
├─sda1
│    ext4         b9ffc3d1-86b2-4a2c-a8be-f2b2f4aa4cb5   11.1G    34% /
├─sda2
│
└─sda5
     swap         f8f6d279-1b63-4310-a668-cb468c9091d8                [SWAP]
sdb
# Ahora tenemos un nuevo dispositivo de bloques; sdb que podremos formatearlo y montarlo.
{% endhighlight %}

Si quisiésemos cerrar la sesión, tenemos que utilizar el parámetro `-u`.

{% highlight bash %}
iscsiadm -m node -T iqn.2020-02.es.luisvazquezalejo:target1 -p 192.168.100.10 -u
{% endhighlight %}

Este nuevo dispositivo de bloques (**/dev/sdb**), podremos gestionarlo como otro cualquiera. Aunque el protocolo que estamos usando es **tcp/ip**, a ojos _Linux_ es un dispositivo que está conectado como podría estarlo un disco **SATA**. Vamos a formatearlo y montarlo.

{% highlight bash %}
root@clienteLinux:/home/vagrant# mkfs.ext4 /dev/sdb
mke2fs 1.44.5 (15-Dec-2018)
Creating filesystem with 512000 1k blocks and 128016 inodes
Filesystem UUID: fddbeb8d-870f-4f5b-add2-b7aefaebd39b
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729, 204801, 221185, 401409

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done 

root@clienteLinux:/home/vagrant# mkdir ISCSI
root@clienteLinux:/home/vagrant# mount /dev/sdb ISCSI/
{% endhighlight %}

Comprobamos que está montado

{% highlight bash %}
lsblk -f
NAME   FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT

...

sdb    ext4         fddbeb8d-870f-4f5b-add2-b7aefaebd39b  382.5M    14% /home/vagrant/ISCSI
{% endhighlight %}

A continuación vamos a automatizar la cargar del _target_ y el montaje del volumen. Para esto vamos a necesitar una unidad de **systemd** que se encargará de montar los volúmenes después de que inicie **open-iscsi.service**. Antes vamos a cargar el target en la configuración de _iscsi_ con los siguientes parámetros.

{% highlight bash %}
iscsiadm --mode node -T iqn.2020-02.es.luisvazquezalejo:target1 -o update -n node.startup -v automatic
{% endhighlight %}

Después nos dirigimos al fichero **/etc/iscsi/iscsiadm.conf** y modificamos el siguiente valor de _manual_ a _automatic_.

{% highlight bash %}
#node.startup = manual
node.startup = automatic
{% endhighlight %}

Después reiniciamos el servicio y observamos como se carga automáticamente.

<p><script id="asciicast-giSMSvYrnWduD85lJaJ8gPcP3" src="https://asciinema.org/a/giSMSvYrnWduD85lJaJ8gPcP3.js" async></script><p>

Por útlimo vamos a crear la unidad de **systemd** tipo **.mount** que mencionamos antes. La sintaxis es bastante sencilla, y es que solo tenemos que indicar **qué** vamos a montar y **dónde** lo vamos a montar. En este ejercicio vamos a crear el directorio **/ISCSI** donde crearemos a su vez, el árbol de directorios en el cual estarán todas las unidades montadas por _ISCSI_ (por lo que ejecutamos `mkdir /ISCSI`).
Ahora sí, creamos la unidad:

{% highlight bash %}
[Unit]
Description=Unidad de montaje para el target1
After=open-iscsi.service

[Mount]
What=/dev/disk/by-uuid/fddbeb8d-870f-4f5b-add2-b7aefaebd39b
Where=/ISCSI/target1
Type=ext4
Options=defaults

[Install]
WantedBy=multi-user.target
{% endhighlight %}

Las unidades de tipo **mount** deben tener un nombre igual al directorio del punto de montaje, solo que cambiamos las **/** por **-**. En este caso se llamará **ISCSI-target1.mount**. Ya solo nos faltaría reiniciar los demonios, habilitar la unidad e iniciarla:

{% highlight bash %}
systemctl daemon-reload
systemctl enable ISCSI-target1.mount
systemctl start ISCSI-target1.mount
{% endhighlight %}

Si probamos a ver el estado del servicio de esta unidad, podemos comprobar que está en **enable** por lo que se ejecutará al inicio del sistema después de **open-iscsi**.

{% highlight bash %}
● ISCSI-target1.mount - Unidad de montaje para el target1
   Loaded: loaded (/etc/systemd/system/ISCSI-target1.mount; enabled; vendor preset: enabled)
   Active: active (mounted) since Tue 2020-02-04 17:39:22 GMT; 4min 15s ago
    Where: /ISCSI/target1
     What: /dev/sdb
    Tasks: 0 (limit: 544)
   Memory: 112.0K
   CGroup: /system.slice/ISCSI-target1.mount

Feb 04 17:39:22 clienteLinux systemd[1]: Mounting Unidad de montaje para el target1...
Feb 04 17:39:22 clienteLinux systemd[1]: Mounted Unidad de montaje para el target1.
{% endhighlight %}

## Caso 2. Target con 2 LUN y cliente Windows CHAP

### Configuración del servidor

Esta vez vamos a crear un _target_ con dos **LUN**. El procedimiento es prácticamente el mismo que en el primer caso por lo que primero creamos el target:

{% highlight bash %}
tgtadm --lld iscsi --op new --mode target --tid 2 -T iqn.2020-02.es.luisvazquezalejo:targetwindows
{% endhighlight %}

Después añadimos las unidades lógicas al target como hicimos antes.

{% highlight bash %}
tgtadm --lld iscsi --op new --mode logicalunit --tid 2 --lun 1 -b /dev/ISCSI/vol2
tgtadm --lld iscsi --op new --mode logicalunit --tid 2 --lun 2 -b /dev/ISCSI/vol3
{% endhighlight %}

Esta vez vamos a establecer una restricción a la hora de acceder al target, y es que activaremos la autenticación **CHAP**. Básicamente consiste en una autenticación _usuario/contraseña_, así que primero creamos al usuario y después lo añadimos al _target_.

{% highlight bash %}
#Creamos al usuario
tgtadm --lld iscsi --mode account --op new --user "Usuario" --password "contrasena1234"
#Lo añadimos al target
tgtadm --lld iscsi --mode account --op bind --tid 2 --user "Usuario"
{% endhighlight %}

Como este _target_ lo utilizaremos en **windows**, tenemos que tener en cuenta que la contraseña tiene que tener obligatoriamente **entre 12 y 15 caracteres**.
Por último creamos la _ACL_ para que sea visible en el _discovery_.

{% highlight bash %}
tgtadm --lld iscsi --op bind --mode target --tid 1 -I ALL
{% endhighlight %}

### Configuración del cliente

Los pasos a seguir en windows son bastante simples. Encendemos la máquina, en el buscador del menú de inicio introducimos "ISCSI" y accedemos al programa "**ISCSI Initiator**"

![](/images/ISCSIwindows/menu)

Después nos dirigimos a la última pestaña, la de configuración y cambiamos el nombre del _initiator_.

![](/images/ISCSIwindows/changeinitiator)

Volvemos a la primera pestaña, refrescamos la página, seleccionamos el _target_ deseado y nos conectamos. Se nos abrirá una ventana, donde tendremos que acceder a la configuración avanzada. Aparecerá una última ventana donde tendremos que dejar marcado la casilla de **enable CHAP log on** e introduciremos el usuario y la contraseña que establecimos antes.

![](/images/ISCSIwindows/logon)

Ahora vamos a darle formato al disco y vamos a probar que podemos acceder. Para hacer esto utilizaremos el programa "**Create and format disk partitions**". En este caso, vamos a crear un **raid 0** y lo formatearemos con **NTFS**.

![](/images/ISCSIwindows/raid0)

![](/images/ISCSIwindows/NTFS)

Por último probamos a acceder a la unidad.

![](/images/ISCSIwindows/access)
