---
title:  "Integración contínua con Jekyll"
excerpt: "Despliegue con Travis-CI en Github Pages"
date:   2020-01-17 13:00:00
categories: [jekyll,travis-ci,github]
tags: [jekyll,integracion-continua]
---

## Introducción

La _integración contínua_ es un modelo de trabajo propuesto por [Martin Fowler](https://es.wikipedia.org/wiki/Martin_Fowler) que consiste en un despliegue y la consiguiente integración de un proyecto de la forma más automática posible. La **integración** en si misma es la compilación y la ejecución de pruebas sobre el resultado obtenido.
Hoy en día existen diversos _servicios_ que nos proporcionan este tipo de integración. Por ejemplo, la página [Travis-CI](https://travis-ci.org) nos da la posibilidad de automatizar estos despliegues a través de [Github](https://github.com/). De hecho podremos programar a través de ficheros **yaml**, donde podremos especificar la ejecución de diversos **script**, sobre qué repositorio y rama(s) se desplegará, etc.

Para registrarnos tan solo tendremos que tener una cuenta en **github**, cuenta a la que se nos pedirá una serie de permisos para poder ejecutar los correspondientes _triggers_ (o _webhooks_ como los llama **github**).

<a href="/images/sign-up-travis.ong"><img src="/images/sign-up-travis.ong" /></a>


## Preparación del repositorio

En entradas anteriores teníamos nuestro proyecto dividido en dos repositorios y dichos repositorios los teníamos en local. Como con *Travis-CI* tanto la construcción de la página como el despliegue se realiza en desde un **docker** remoto, no necesitaremos tener el repositorio contenedor del html en local. De hecho, para tener un mayor orden vamos a situar todo en un mismo repositorio dividido en dos ramas distintas.
La rama **master** contendrá los ficheros necesarios para que **jekyll** pueda construir la página. Y la rama **gh-pages** será la contenedora del *html*, y a la que estará "apuntando" el servicio de **Github Pages**.

En mi caso el repositorio lo llamaré **blog**


## Configuración Travis-CI

Una vez nos hemos registrado, tendremos que dirigirnos al apartado de nuestro perfil, donde veremos un listado de nuestros repositorios. En esta ocasión vamos a activar solo el repositorio donde tenemos los ficheros **jekyll**.
