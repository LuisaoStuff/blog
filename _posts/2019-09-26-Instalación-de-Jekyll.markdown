---
title:  "Instalación y configuración de Jekyll!"
date:   2019-09-25 15:04:23
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

