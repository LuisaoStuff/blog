---
title:  "Publicación con Github-Pages"
excerpt: "Crea un repositorio y conviértelo en tu página"
date:   2019-10-02 10:57:23
categories: [jekyll, github]
tags: [jekyll, github]
---
<a href="https://imgur.com/DIYxa6F"><img src="https://imgur.com/DIYxa6F.png" title="source: imgur.com" /></a>
**Github** es una plataforma de desarrollo de software colaborativo y de control de versiones. Aunque existen algunas versiones paralelas como **Gitlab**, que también se basan en el paquete *git*, esta vez nos centraremos en **Github**.
Lo primero que debemos tener es una cuenta en dicha página, además de contar con el paquete *git* instalado en nuestra máquina. Para instalar este paquete solo tenemos que ejecutar `$ apt install git`. Una vez hecho esto, accedemos a nuestra cuenta en github y creamos un nuevo repositorio.
<a href="https://imgur.com/dYfEZq8"><img src="https://imgur.com/dYfEZq8.png" title="source: imgur.com" /></a>
A la hora de nombrar el repositorio tenemos dos opciones:
* Página de *usuario* (solo dispones de una): **nombre-de-usuario.github.io**
* Página de *proyecto* (dispones de una por repositorio): **nombre-de-usuario.github.io/repositorio**
En mi caso ya no puedo crear otra página de usuario, por lo que crearé la página *"prueba"*

<a href="https://imgur.com/s8KyXwu"><img src="https://imgur.com/s8KyXwu.png" title="source: imgur.com" /></a>

## Inicializar repositorio en la máquina local

Copiamos la dirección *ssh* que obtenemos al iniciar el repositorio:
<a href="https://imgur.com/mMNICQJ"><img src="https://imgur.com/mMNICQJ.png" title="source: imgur.com" /></a>
Después entramos en el directorio donde se genera el html, normalmente en el directirio **_site/** y procedemos a ejecutar los siguientes comandos:
```
$ git init
$ git add .
$ git commit -m “first commit”
$ git remote add origin git@github.com:LuisaoStuff/luisaostuff.github.io-prueba.git
$ git push -u origin master
```
A partir de aquí, cada vez que queramos modificar la página, deberíamos generarla en local para que se actualicen los ficheros html y después añadirlos y subirlos con `git push`.

## Integración continua

Estoy probando la integración contínua con bash y hookkkkkk
