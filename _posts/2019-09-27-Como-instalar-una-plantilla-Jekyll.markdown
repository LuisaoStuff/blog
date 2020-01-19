---
title:  "Como instalar una plantilla Jekyll"
excerpt: "Personaliza tu página!"
date:   2019-09-27 10:57:23
categories: [jekyll]
tags: [jekyll]
---

Lo primero que debemos hacer es buscar una plantilla que nos guste. Para esto hay diversas páginas, por ejemplo [Jekyll Themes](http://jekyllthemes.org/). Una vez hemos encontrado algo que encaje con nuestra idea, descargamos el **zip** que nos ofrezca.

<a href="/images/download-theme.png"><img src="/images/download-theme.png" /></a>
Lo descomprimimos y empezamos con la instalación de las _gemas_. Accedemos al directorio, en nuestro caso **./jekyll_uno** y ejecutamos `$ bundle install`.
En la distribución de **debian** suele dar problemas el paquete **nokogiri** al intentar instalarlo con el gestor *bundle*. La solución más habitual a este problema de dependencias, es seguir los siguientes pasos que puedes ver en esta [página](https://nokogiri.org/tutorials/installing_nokogiri.html).

Ya solo nos quedaría inicializar jekyll y acceder a la página web para ver el resultado.
<a href="/images/test-j.png"><img src="/images/test-j.png" /></a>
