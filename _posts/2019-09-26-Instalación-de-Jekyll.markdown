---
title:  "Instalación y configuración de Jekyll"
excerpt: "Pequeña guía desde 0"
date:   2019-09-25 15:04:00
categories: [jekyll]
tags: [jekyll]
---

## Instalar Jekyll

Para instalar *jekyll* en Debian primero debemos comprobar que tenemos instalado *Ruby*, pues es el lenguaje que utiliza. Una vez hecho esto (y tras haber *actualizado* el sistema) ejecutamos el siguiente comando:
```
$ gem install jekyll bundler
```

## Crear nueva página

Es bastante sencillo, tan solo hay que ejecutar `$ jekyll new jekyll-site` y automáticamente se generará los correspondientes ficheros de configuración y los directorios donde almacenaremos los *markdown* que darán contenido a nuestra página.Esta es la estructura generada:
```
$ tree jekyll-site
jekyll-site/
├── 404.html
├── about.md
├── _config.yml
├── Gemfile
├── Gemfile.lock
├── index.md
└── _posts
    └── 2019-09-25-welcome-to-jekyll.markdown

1 directory, 7 files
```

### Ficheros y directorios importantes

* `Gemfile`: Te permite **definir las dependencias** y las librerías de **Ruby**, y es similar al fichero _requeriments.txt_ que usábamos en *python* a la hora de desplegar la aplicación en *heroku*.
* `_config.yml`: Es el **fichero de configuración** de jekyll (puedes modificar cómo va a generar las páginas).
* `/posts`: Es el **directorio donde vamos a guardar** los ficheros **markdown** (en orden _cronológico_) que darán contenido a nuestra web.

## Lanzar el entorno web


Para generar la web en el entorno de desarrollo, entramos en el directorio `/jekyll-site` y ejecutamos: `$ bundle exec jekyll serve` .De esta manera podremos tener una previsualización de la web accediendo a _localhost:4000_ desde el navegador.

## Modificando _config.yml

Este fichero contiene por defecto solo algunos parámetros, pero podemos añadir cuantos nosotros queramos. Si instalamos una plantilla (como explico [aquí](/Como-instalar-una-plantilla-Jekyll/)) podremos observar que hay multiples parámetros adicionales. En principio, estos son los parámetros básicos:
```
# Site settings
title: Your awesome title
email: your-email@example.com
baseurl: "/"

# Build settings
markdown: kramdown
theme: minima
plugins:
  - jekyll-feed

```
* `title`: Es el título que tendrá tu página y que aparecerá en el texto de la pestaña.
* `email`: Añade un botón con el email que indiques (de la misma forma se puede añadir *twitter*, *github*, etc).
* `baseurl`: Indica la dirección de la página principal de la web, por ejemplo la raiz **"/"** o **"/home"**.
* `markdown`: Se especifica la librería de *Ruby* que se usará para **generar** el **html** a partir del **markdown**.

### Ejemplo básico

```
title: El título funciona!
email: luisvazquezalejo@gmail.com
baseurl: "/home"
github_username:  luisaostuff

# Build settings
markdown: kramdown
theme: minima
plugins:
  - jekyll-feed
```
<a href="/images/local-deploy.png"><img src="/images/local-deploy.png" /></a>

## Creando la primera página

Para añadir páginas, deberás escribirlas en formato *markdown* y situarlas en el directorio **/_posts**. Nuestro ejemplo consistirá en una *entrada de blog* y modificaremos algunas lineas para dejar claras las reglas que deberemos seguir.
Lo primero que veremos en el fichero es una **cabecera** como esta:
```
---
layout: post
title:  "Mi primera entrada!"
date:   2019-10-02 23:18:39 +0200
categories: ejemplo
tags: [ejemplo, tutorial]
---
```
Esta cabecera define los **datos principales**, como el título de la entrada, la fecha y la categoría. Todo esto es **necesario para** jekyll a la hora de **organizar** el contenido **de forma automática**. Por ejemplo, también indicaremos en el nombre del fichero alguno de estos parámetros, siguiendo este formato: **YYYY-MM-DD-nombre-del-post.markdown**.
<a href="/images/first-page.png"><img src="/images/first-page.png" /></a>

### Enlaces de interés:

* [markdown.es](https://markdown.es/) 
