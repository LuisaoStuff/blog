---
title:  "Introducción a Postfix"
excerpt: "Configuración del servidor de correos Postfix con relayhost"
date:   2020-02-24 14:25:00
categories: [postfix,Servicios]
---

## Introducción

Postfix es un **servidor de correos** cuya primera versión data de **2001**. Aunque su origen nos pueda parecer ya lejano, este _software_ se ha mantenido y se ha ido desarrollando a lo largo de los años, siendo su última **versión estable** la **3.4.9**, la cual fue lanzada hace muy poco (3 de febrero de 2020).
Aunque en sus inicios, nos bastaba con el servidor de correos (utilizandose simplemente en local, entre usuarios del sistema), hoy en día el esquema de funcionamiento estandar se compone de tres bloques

![](/images/escenario-postfix/esquema.png)

* **MUA**: Son los clientes de correo, por ejemplo _**thunderbird**_, que son los que normalmente utilizan los usuarios finales. Se encargan de enviar el correo al **MTA**, y lo hacen a través del protocolo **SMTP**, mientras que cuando lo descargan, utilizan el protocolo **POP** o **IMAP**.

* **MTA**: Los propios servidores de correo. Pueden tener varios _roles_; servidor destinatario, servidor remitente, pueden tener en la misma máquina un **MDA** o consultar desde la misma máquina el buzón a través de linea de comandos.

* **MDA**: Se encargan, utilizando el protocolo [POP](https://es.wikipedia.org/wiki/Protocolo_de_oficina_de_correo) o [IMAP](https://es.wikipedia.org/wiki/Protocolo_de_acceso_a_mensajes_de_Internet), de servir el correo a los usuarios finales, ya sean clientes web ([gmail](mail.google.com)) o clientes aplicaciones de escritorio
    * Protocolo **POP**: Descarga el correo del servidor en el cliente, y borra dichos correos del servidor. En el momento en el que disponemos de diversos clientes, este protocolo es poco útil.
    * Procotolo **IMAP**: Permite visualizar los correos de forma remota, por lo que no los borra de la fuente y podemos ir sincronizando cada cliente con el servidor de correos. Este es el protocolo que utilizan la mayoría de clientes web, como **gmail**.

En esta práctica, instalaremos el servidor de correos **Postfix** en nuestra máquina _croqueta_ alojada en una nube privada [OpenStack](https://www.openstack.org/) que además contiene otros servicios como un **DNS**.

## Postfix como servidor SMTP

Antes de seguir, vamos a habilitar en el grupo de seguridad general de **openstack** el puerto **25** (más tarde habilitaremos el protocolo **SMTPS**, el puerto **465**). Una vez hecho esto, instalamos **postfix** con **apt**:

{% highlight bash %}
apt install postfix
{% endhighlight %}

Durante la instalación nos pedirá una serie de parámetros, como el _system mail name_ (tu dominio, el mío es **luis.gonzalonazareno.org**) y el tipo de servidor, el cual seleccionaremos como **local**.
En mi caso, como en la mayoría, los correos de esta red solo podrán ser enviados por un único servidor de correos. Esto se debe al gran número de máquinas que se conectan en la red local y lo facil que sería que alguna se infectase con algún virus y empezase a enviar _spam_ de forma indiscriminada, provocando que la IP del instituto acabase en una [lista negra de correo](https://www.nerion.es/blog/listas-negras-de-spam-que-son-y-como-funcionan/).
Después de contactar con el administrador para que añada mi servidor de correo a la lista de _relay hosts_ de la máquina **babuino-smtp**, empiezo a configurar mi servidor **postfix** para que pueda enviar correos al exterior. Para hacer esto nos dirigimos al fichero **/etc/postfix/main.cf** y modificamos la directiva `relayhost`:

{% highlight bash %}
relayhost = babuino-smtp.gonzalonazareno.org
{% highlight bash %}

Vamos a probar a enviar un correo a **gmail** con el comando `mail`.

{% highlight bash %}
root@croqueta:~# mail luisvazquezalejo@gmail.com # Destinatario
Cc: 						 # Destinatarios adicionales
Subject: prueba					 # Asunto
esto es una prueba de relay			 # Cuerpo del mensaje
^D						 # Fin del mensaje
{% highlight bash %}

Comprobamos la bandeja de entrada, y efectivamente tenemos un mensaje de **root@luis.gonzalonazareno.org**.

![](/images/escenario-postfix/relay-test.png)

Si quisiésemos poder responder a los correos llegados del dominio **luis.gonzalonazareno.org**, tendríamos que definir un registro en nuestro **DNS** del tipo **MX**. Como dijimos antes, en la máquina _Croqueta_, tenemos instalado un servidor **DNS**, concretamente **bind9**, por lo que nos dirigimos a los ficheros contenedores de las zonas dns y añadimos la siguiente linea en ambos ficheros:

* **/var/cache/bind/db.interna.luis.gonzalonazareno.org**
* **/var/cache/bind/db.externa.luis.gonzalonazareno.org**

{% highlight bash %}
$ORIGIN luis.gonzalonazareno.org.
@		IN	MX	10 croqueta
{% endhighlight %}

Intentamos responder el mensaje de **root** que enviamos antes.

![](/images/escenario-postfix/answer-gmail.png)

Observamos el log

{% highlight bash %}
tail -f /var/log/mail.log
Feb 24 11:14:43 luis postfix/smtpd[3320]: connect from babuino-smtp.gonzalonazareno.org[192.168.203.3]
Feb 24 11:14:43 luis postfix/smtpd[3320]: 3BB602121A: client=babuino-smtp.gonzalonazareno.org[192.168.203.3]
Feb 24 11:14:43 luis postfix/cleanup[3325]: 3BB602121A: message-id=<CAO8TDyehWQ3P9ebF6ma-0c0Ow-PDsQUmgQmyP77qqVcyN9mCLg@mail.gmail.com>
Feb 24 11:14:43 luis postfix/qmgr[2751]: 3BB602121A: from=<luisvazquezalejo@gmail.com>, size=3720, nrcpt=1 (queue active)
Feb 24 11:14:43 luis postfix/smtpd[3320]: disconnect from babuino-smtp.gonzalonazareno.org[192.168.203.3] ehlo=1 mail=1 rcpt=1 data=1 quit=1 commands=5
Feb 24 11:14:43 luis postfix/local[3326]: 3BB602121A: to=<root@luis.gonzalonazareno.org>, relay=local, delay=0.09, delays=0.04/0.04/0/0.01, dsn=2.0.0, status=sent (delivered to mailbox)
{% endhighlight %}

Si queremos consultar la bandeja de entrada de un usuario, ejecutamos el comando `mail` y seleccionamos el número del correo.

{% highlight bash %}
root@croqueta:~# mail
"/var/mail/root": 7 messages 1 new 6 unread

...

>N   7 Luis Vázquez Alej Mon Feb 24 11:14  72/3782  Re: prueba
? 7
Return-Path: <luisvazquezalejo@gmail.com>
X-Original-To: root@luis.gonzalonazareno.org
Delivered-To: root@luis.gonzalonazareno.org
Received: from babuino-smtp.gonzalonazareno.org (babuino-smtp.gonzalonazareno.org [192.168.203.3])
	by luis.gonzalonazareno.org (Postfix) with ESMTP id 3BB602121A
	for <root@luis.gonzalonazareno.org>; Mon, 24 Feb 2020 11:14:43 +0000 (UTC)
Received: from mail-il1-f175.google.com (mail-il1-f175.google.com [209.85.166.175])
	by babuino-smtp.gonzalonazareno.org (Postfix) with ESMTPS id D230ABD9D4
	for <root@luis.gonzalonazareno.org>; Mon, 24 Feb 2020 11:14:42 +0000 (UTC)
Received: by mail-il1-f175.google.com with SMTP id f70so7380206ill.6
        for <root@luis.gonzalonazareno.org>; Mon, 24 Feb 2020 03:14:42 -0800 (PST)
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
        d=gmail.com; s=20161025;
        h=mime-version:references:in-reply-to:from:date:message-id:subject:to;
        bh=EcyVo515TGRGaicW22HJHAcUCBv6Lj8I38/5keF7Y5Q=;
        b=IqQC7UyfHaJJOJEL2+wdYsi3N+o5q7vPes1O4sXc14sqIj1rbqrov/aK+yjt+mpcOB
         fnbpC7cpXCos8TzOqQQF8NYT1ZFQaRa2yL5N8JwamuEkztsZKH0aJ0nDOt9zdK7qSBRj
         ae1//p7zPuu9yKZb82YfNSJ1drXpQ3CcUsB8H0dYwYTQSw8VMLozIw+bpPZBnU8RJ8Na
         8JLjHF2ELFZtTYaz5UKfKzuFxKKW4M4QK0kqL4E82fdBejB3c7WefMCffqh0QrY7uysC
         7uKVLA9kCRocXIYtDPe6TAp5bnjYTNErDQEcp3Sg5fTEGkiWaFeEGwXQ5HtH+Q03WFiP
         ZdWA==
X-Google-DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
        d=1e100.net; s=20161025;
        h=x-gm-message-state:mime-version:references:in-reply-to:from:date
         :message-id:subject:to;
        bh=EcyVo5
...
{% endhighlight %}

## Dovecot como servidor MDA

La instalación de esta aplicación es bastante sencilla. Se compone de tres paquetes, que instalaremos con **apt**.

{% highlight bash %}
apt -y install dovecot-core dovecot-pop3d dovecot-imapd
{% endhighlight %}

Y lo único que tendremos que modificar de los ficheros de configuración será descomentar la linea `listen = *, ::` del fichero **/etc/dovecot/dovecot.conf**, modificar la directiva `mail_location` al valor **maildir:~/Maildir**, y añadimos la linea `disable_plaintext_auth = no` al fichero **/etc/dovecot/conf.d/10-auth.conf**.
Después vamos a crear dos reglas nuevas en **openstack** para permitir el trafico en los puertos **110** y **143**.
Como cliente vamos a utilizar **evolution**, y para añadir una cuenta de correo tenemos que seguir los siguientes pasos. Primero nos dirigimos a la pestaña **editar** y pulsamos sobre **preferencias**. Una vez se nos ha abierto la ventana, hacemos _click_ sobre **añadir**.
Seguimos los siguientes pasos. En mi caso, el usuario del sistema al que enviaré el correo es **debian**, la máquina es **smtp.luis.gonzalonazareno.org** y el protocolo que voy a utilizar en primera instancia es **imap** sin ningún tipo de cifrado.

![](/images/escenario-postfix/add-account1.png)

![](/images/escenario-postfix/add-account2.png)

![](/images/escenario-postfix/add-account3.png)

Probamos a enviar un correo a **gmail.

![](/images/escenario-postfix/send-from-evolution.png)

Y miramos que nos ha llegado.

![](/images/escenario-postfix/evolution-test.png)

Ahora vamos a responder el mensaje y comprobamos qué pasa cuando lo descargamos primero con el protocolo **IMAP** y luego con **POP3**.

![](/images/escenario-postfix/imap-download.png)

Si comprobamos el directorio de correos de **debian**, podemos observar que se conservan los ficheros locales.

{% highlight bash %}
debian@croqueta:~/Maildir/cur$ ls
1582551516.Vfe01I41f38M826440.croqueta:2,S  1582618055.Vfe01I41f2bM897460.croqueta:2,S
{% endhighlight %}

Pero si cambiamos el protocolo en las preferencias, y utilizamos **POP3**, vamos a comprobar que se borran los ficheros del directorio local.

![](/images/escenario-postfix/pop-config.png)

Si volvemos a comprobar los correos tras descargarlos en Evolution, podemos observar que ya no se encuentran en el directorio **~/Maildir**.

{% highlight bash %}
debian@croqueta:~$ ls Maildir/cur/
debian@croqueta:~$ 
{% endhighlight %}

## Correo con crontab

Para poder enviar correos con las tareas de **cron**, tendremos que especificar en el fichero la directiva `MAILTO = usuario`. Para acceder a dicho fichero ejecutamos `crontab -e` (la primera vez nos pedirá que indiquemos el editor de texto que vamos a usar).
Una vez dentro del fichero indicamos la directiva y definimos una tarea de cron. En mi caso simplemente voy a ejecutar un _script_ que realiza un `ls` sobre el directorio de **/root/**. El contenido del fichero debería ser algo como esto:

{% highlight bash %}
MAILTO = root

* * * * * /root/cron.bash
{% endhighlight %}

Vamos a ir un paso más allá y vamos a hacer que ese correo nos llegue a nuestro correo personal. Primero vamos a definir en el fichero **/etc/aliases** un alias para que se reenvíe el correo de **root** a **debian**. Añadimos la siguiente linea:

{% highlight bash %}
root: debian
{% endhighlight %}

Y por último creamos un fichero **.forward** en el directorio _home_ de debian, que contendrá nuestra dirección de correo personal:

{% highlight bash %}
debian@croqueta:~$ nano .forward 

	luisvazquezalejo@gmail.com
{% endhighlight %}

Reiniciamos el servicio de **cron** y esperamos a que nos llegue el correo:

{% highlight bash %}
systemctl restart cron
{% endhighlight %}
![](/images/escenario-postfix/crontab-forward-mail.png)


#### Referencias 
* [Documentación de Alberto Molina y José Domingo Muñoz](https://www.google.com/url?sa=i&url=https%3A%2F%2Fplataforma.josedomingo.org%2Fpledin%2Fcursos%2Fservicios2011%2Ffiles%2Fcorreo-e.pdf&psig=AOvVaw19ByLobCmNflwnKNw6OEOc&ust=1581579808920000&source=images&cd=vfe&ved=0CA0QjhxqFwoTCIiar5HCy-cCFQAAAAAdAAAAABAc)
* [Wikipedia](https://es.wikipedia.org/wiki/Postfix)
* [Tarea de José Domingo Muñoz](https://fp.josedomingo.org/serviciosgs/u07/practica_correo.html)
