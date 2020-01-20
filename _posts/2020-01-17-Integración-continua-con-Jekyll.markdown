---
title:  "Integración continua con Jekyll"
excerpt: "Despliegue con Travis-CI en Github Pages"
date:   2020-01-17 13:00:00
categories: [jekyll,travis-ci,github]
tags: [jekyll,integracion-continua]
---

## Introducción

La _integración continua_ es un modelo de trabajo propuesto por [Martin Fowler](https://es.wikipedia.org/wiki/Martin_Fowler) que consiste en un despliegue y la consiguiente integración de un proyecto de la forma más automática posible. La **integración** en si misma es la compilación y la ejecución de pruebas sobre el resultado obtenido.
Hoy en día existen diversos _servicios_ que nos proporcionan este tipo de integración. Por ejemplo, la página [Travis-CI](https://travis-ci.org) nos da la posibilidad de automatizar estos despliegues a través de [Github](https://github.com/). De hecho podremos programar a través de ficheros **yaml**, donde podremos especificar la ejecución de diversos **script**, sobre qué repositorio y rama(s) se desplegará, etc.

Para registrarnos tan solo tendremos que tener una cuenta en **github**, cuenta a la que se nos pedirá una serie de permisos para poder ejecutar los correspondientes _triggers_ (o _webhooks_ como los llama **github**).

<a href="/images/sign-up-travis.png"><img src="/images/sign-up-travis.png" /></a>


## Preparación del repositorio

En entradas anteriores teníamos nuestro proyecto dividido en dos repositorios y dichos repositorios los teníamos en local. Como con *Travis-CI* tanto la construcción de la página como el despliegue se realiza en desde un **docker** remoto, no necesitaremos tener el repositorio contenedor del html en local. De hecho, para tener un mayor orden vamos a situar todo en un mismo repositorio dividido en dos ramas distintas.
La rama **master** contendrá los ficheros necesarios para que **jekyll** pueda construir la página. Y la rama **gh-pages** será la contenedora del *html*, y a la que estará "apuntando" el servicio de **Github Pages**.

En mi caso el repositorio lo llamaré **blog** y además de contener los ficheros habituales de jekyll, vamos a añadir un fichero *yaml* con la siguiente configuración:

{% highlight ruby %}
language: ruby
rvm:
  - 2.6.3

script: ./script/cibuild		# Ejecutamos el script para construir
					# el html
branches:
  only:
  - master				# Estas son las ramas donde va a
  - gh-pages				# actuar Travis

env:
  global:
  - NOKOGIRI_USE_SYSTEM_LIBRARIES=true	# Con esto aceleramos la instalación 
					# de html-proofer

addons:
  apt:
    packages:
    - libcurl4-openssl-dev

cache: bundler 				# Guardamos en caché la instalación
					# de gemas

notifications:				# Desactivamos las notificaciones por
  email: false				# mail

deploy:					# Desplegamos usando el proveedor pages
  provider: pages			# de github-pages. Especificamos el
  skip_cleanup: true			# token $GITHUB_TOKEN que definiremos
  local_dir: _site			# más adelante en el dashboard.
  github_token: $GITHUB_TOKEN		# Con la opción "repo" y "branch"
  on:					# indicamos el repositorio y la rama
    repo: LuisaoStuff/blog		# respectivamente.
    on:					# Por último, con el parámetro "fqdn"
      branch: gh-pages			# definimos el dominio personal que
  fqdn: blog.luisvazquezalejo.es	# vamos a utilizar.
{% endhighlight %}

El contenido del *script* cuya ejecución hemos definido en el fichero *yaml*, tendrá el siguiente contenido:

{% highlight bash %}
#!/usr/bin/env bash
# Levanta una excepción en caso de error
set -e

# Construímos el html incrementalmente
bundle exec jekyll build --incremental

# Verificamos el html y comprobamos que las url externas son https
bundle exec htmlproofer --empty-alt-ignore --enforce-https ./_site
{% endhighlight %}

## Configuración Travis-CI

Una vez nos hemos registrado, tendremos que dirigirnos al apartado de nuestro perfil, donde veremos un listado de nuestros repositorios. Como hemos indicado antes, vamos a trabajar sobre el repositorio **blog**, por lo tanto lo activamos.

<a href="/images/activate-repository.png"><img src="/images/activate-repository.png" /></a>

Vamos a necesitar un **access token**, por lo que nos dirigimos a nuestra cuenta de *github* y seguimos estos pasos. Accedemos a *settings*

<a href="/images/settings.png"><img src="/images/settings.png" /></a>

*Developer settings*

<a href="/images/developer-settings.png"><img src="/images/developer-settings.png" /></a>

*Personal access token*

<a href="/images/Personal-access-token.png"><img src="/images/Personal-access-token.png" /></a>

Marcamos las siguientes *opciones*

<a href="/images/token-options.png"><img src="/images/token-options.png" /></a>

Copiamos el código generado y lo introducimos en Travis.

<a href="/images/token-code.png"><img src="/images/token-code.png" /></a>

<a href="/images/travis-token.png"><img src="/images/travis-token.png" /></a>
