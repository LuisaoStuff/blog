---
title:  "Introducción a Postfix"
excerpt: "Configuración del servidor de correos Postfix con relayhost"
date:   2020-02-25 14:25:00
categories: [postfix,Servicios]
---

## Introducción

Postfix es un **servidor de correos** cuya primera versión data de **2001**. Aunque su origen nos pueda parecer ya lejano, este _software_ se ha mantenido y se ha ido desarrollando a lo largo de los años, siendo su última **versión estable** la **3.4.9**, la cual fue lanzada hace muy poco (3 de febrero de 2020).
Aunque en sus inicios, nos bastaba con el servidor de correos (utilizandose simplemente en local, entre usuarios del sistema), hoy en día el esquema de funcionamiento estandar se compone de tres bloques

![](/images/escenario-postfix/esquema.png)

* **MUA**: Son los clientes de correo, por ejemplo _**thunderbird**_, que son los que normalmente utilizan los usuarios finales. Se encargan de enviar el correo al **MTA**, y lo hacen a través del protocolo **SMTP**, mientras que cuando lo descargan, utilizan el protocolo **POP** o **IMAP**.

* **


* **Referencias**: [Documentación de Alberto Molina y José Domingo Muñoz](https://www.google.com/url?sa=i&url=https%3A%2F%2Fplataforma.josedomingo.org%2Fpledin%2Fcursos%2Fservicios2011%2Ffiles%2Fcorreo-e.pdf&psig=AOvVaw19ByLobCmNflwnKNw6OEOc&ust=1581579808920000&source=images&cd=vfe&ved=0CA0QjhxqFwoTCIiar5HCy-cCFQAAAAAdAAAAABAc), [Wikipedia](https://es.wikipedia.org/wiki/Postfix)
