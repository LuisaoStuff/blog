---
title:  "Publicación con Github-Pages PRUEBA"
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
A partir de aquí, cada vez que queramos modificar la página, deberíamos generarla en local para que se actualicen los ficheros html y después añadirlos y subirlos con `git push`. Además indicaremos en el repositorio del entorno desarrollo que no se suban los ficheros **.html** ubicados en el directorio **_site**. Esto lo indicamos en el fichero **.gitignore**

## Integración continua

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



b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEA+Q1/RhkMDqGlcCmq8xJUKZEJ3gy3J/WeiDbHzJvVk9RMf30ycFvb
JW4WqVNKyQzeuIYRRFVfUr7pMc40UCtxKqBmXCFsr8Wb4hKmlS0hTRLb2z1BuEWx7kqMJS
lKc/W1bvZ+24VsWOJiMa1z5OIBfiCA0Dnuqmb9nTXHlxtT0Ek0lni+ysdi0sa7yA2L+X9z
eNtivuw2he1965g+r14XtZ4bcNnQ6bXsHXhjuFGM3Mn8n9sa0pqE+nqLFqdbyptQvFelaa
P1bwXjv+2618xqmHj8hOgctHBvWwxfXFRVy4UZw5jRTAbUaoyuo6uRqqbklokMc7JUOLIS
4Q6DIxTElwAAA8h9zHrDfcx6wwAAAAdzc2gtcnNhAAABAQD5DX9GGQwOoaVwKarzElQpkQ
neDLcn9Z6INsfMm9WT1Ex/fTJwW9slbhapU0rJDN64hhFEVV9SvukxzjRQK3EqoGZcIWyv
xZviEqaVLSFNEtvbPUG4RbHuSowlKUpz9bVu9n7bhWxY4mIxrXPk4gF+IIDQOe6qZv2dNc
eXG1PQSTSWeL7Kx2LSxrvIDYv5f3N422K+7DaF7X3rmD6vXhe1nhtw2dDptewdeGO4UYzc
yfyf2xrSmoT6eosWp1vKm1C8V6Vpo/VvBeO/7brXzGqYePyE6By0cG9bDF9cVFXLhRnDmN
FMBtRqjK6jq5GqpuSWiQxzslQ4shLhDoMjFMSXAAAAAwEAAQAAAQBDM9UoARIz0IJnpZav
SD7ViIF1HVE+wxQoBUAcgeA7p4mMzSeTEfYsP2x1/Det0H84o1R9b9vs4/7gpZeQGmjK68
UzDwHY3CWX9xhkIG1f8rrIidr18jh06ECwtleUurReYL0SVwpJYazFYtxm4mUst3CKv1cb
O/crOJvGtmUDSnVVHqxcPucQXWK4OvyM8sWq8i6keD5LvX//c9KDNdIDxDogq7iXb+iCOd
DVOJzVxIJ9xLzRghCrKmSMTYEeu5EWnB4fFIEMMUy1Dj1o0jMJDtggTeGBFDijIPuTWgOr
PFEisAoSQFlKCodadbsd19XdJnif6+a7XvR/iiLJ5svBAAAAgQC32FQH77nqCBdN8G3L69
7YkFVw7QNpREO6ANlplX6lwMMI+diCAr/tijukgHXP+zLMQqKQl75B6frXUUMvYi4B7Ulb
y/RvIgQDaR/0YHkepACEFhY+JERpRGz0o3+e+SM75UFy9eVv3ZSzi1bF6FlGU3W2fhJ/cd
q9T8418qe81gAAAIEA/YoBEj0x5hK6e3Fv0AHtNouG0VkJFOQbpdEii63PqdAGMRuFrfAg
TcxAO/V9JUM3aDy5uqeSESHA5P84QmylJiKyajFekimiMECC1wKeXhWe748xsRM8xW8w33
Dhwy6k5IO3IHYVJxsGuUomj2CW5Ggi6Hudi7Jot9ju2jpc338AAACBAPt4WGI31i18/KFO
fBRykn1Z2Oq1LQletdJbX9cnMKmh3pcrB+ouj4oE+20We26hQSkr14I97SXFsBk5+Ux+fp
ggM0NHwG9c+xxjaBMns2g+fwGQlKKnqYjPDhHk/z09+A9HKSKWNoiBjmR0uCyuqNFDeMKn
PNDB9m/Xcejk36bpAAAAC2x1aXNAa3V0dWx1AQIDBAUGBw==
