---
title:  "Publicación con Github-Pages"
excerpt: "Crea un repositorio y conviértelo en tu página"
date:   2019-10-02 10:57:20
categories: [jekyll, github]
tags: [jekyll, github]
---
<a href="/images/git-logo.png"><img src="/images/git-logo.png" /></a>
**Github** es una plataforma de desarrollo de software colaborativo y de control de versiones. Aunque existen algunas versiones paralelas como **Gitlab**, que también se basan en el paquete *git*, esta vez nos centraremos en **Github**.
Lo primero que debemos tener es una cuenta en dicha página, además de contar con el paquete *git* instalado en nuestra máquina. Para instalar este paquete solo tenemos que ejecutar `$ apt install git`. Una vez hecho esto, accedemos a nuestra cuenta en github y creamos un nuevo repositorio.
<a href="/images/new-repository.png"><img src="/images/new-repository.png" /></a>
A la hora de nombrar el repositorio tenemos dos opciones:
* Página de *usuario* (solo dispones de una): **nombre-de-usuario.github.io**
* Página de *proyecto* (dispones de una por repositorio): **nombre-de-usuario.github.io/repositorio**
En mi caso ya no puedo crear otra página de usuario, por lo que crearé la página *"prueba"*

<a href="/images/github-page.png"><img src="/images/github-page.png" /></a>

## Inicializar repositorio en la máquina local

Copiamos la dirección *ssh* que obtenemos al iniciar el repositorio:
<a href="/images/copy-ssh-key.png"><img src="/images/copy-ssh-key.png" /></a>
Después entramos en el directorio donde se genera el html, normalmente en el directirio **_site/** y procedemos a ejecutar los siguientes comandos:
```
$ git init
$ git add .
$ git commit -m “first commit”
$ git remote add origin git@github.com:LuisaoStuff/luisaostuff.github.io-prueba.git
$ git push -u origin master
```
A partir de aquí, cada vez que queramos modificar la página, deberíamos generarla en local para que se actualicen los ficheros html y después añadirlos y subirlos con `git push`. Además indicaremos en el repositorio del entorno desarrollo que no se suban los ficheros **.html** ubicados en el directorio **_site**. Esto lo indicamos en el fichero **.gitignore**

## Despliegue continuo

Como hemos visto, tenemos la página en dos repositorios github, uno con el entorno de desarrollo, donde tenemos los ficheros de configuración y el otro con los ficheros **.html** que componen la página. A continuación vamos a automatizar el proceso de despliegue en producción a través de *Git Hooks*. ¿En qué consisten? Son unos **script** que se comportan como *triggers*, ya que se ejecutan en determinados momentos, como por ejemplo en la ejecución de un **commit** o antes de un **push**. Estos *script* se encuentran dentro del repositorio, concretamente en **.git/hooks/**.
En nuestro caso utilizaremos el fichero **pre-push** que se ejecuta antes del *git push*.
```
echo "actualizando repositorio local"
bundle exec jekyll build --incremental
cp -r /home/luis/Escritorio/github/pagina-jekyll/_site/* /home/luis/Escritorio/github/LuisaoStuff.github.io/
cd /home/luis/Escritorio/github/LuisaoStuff.github.io/
git add *
git commit -m "Autodeploy"
git push
echo "repositorio desplegado en producción"
exit 0
```
Con esto, cada vez que hayamos cambiado algun post, solo tenemos que hacer un `git commit -am` y un `git push` para desplegarlo en producción.
