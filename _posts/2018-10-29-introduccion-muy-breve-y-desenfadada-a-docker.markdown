---
layout: post
title:  "Introducción, muy breve y desenfadada, a Docker"
date:    2018-10-29 00:00:00 +0200
tags: docker
---

Desde que el mundo es mundo han existido batallas entre distintos clanes, entre distintas civilizaciones. Como si de la [guerra entre vampiros y licántropos](https://www.imdb.com/title/tt0320691/) se tratara, en la informática también existe dicha guerra encarnizada: los desarrolladores contra los administradores de sistemas. Hoy os traigo una breve introducción a Docker, el elegido para apaciguar a ambas facciones y traer la paz.

## ¿Qué es Docker?

> Docker es un proyecto de código abierto que automatiza el despliegue de aplicaciones dentro de contenedores de software, proporcionando una capa adicional de abstracción y automatización de virtualización de aplicaciones en múltiples sistemas operativos (Wikipedia).

![morpheo](/assets/images/morpheo.jpg){: .w-25}

Vamos, que lo que nos hace es «contener» servicios, pudiéndolos tener separados entre sí.

## ¿Para qué se usa?

- Para correr servicios abstrayéndose del sistema operativo.
- Para escalar servicios de manera automática.
- Para hacer **desarrollo ágil.**
- **Integración continua.**
- Para probar nuevas versiones de software/librerías de manera más sencilla
  - El desarrollo que tengo en PHP 5.6 va a funcionar den 7.3?
  - Qué tal funciona el nuevo MySQL 8?
  - Pasar de Tomcat 6 a Tomcat 9 va a ser un infierno?

## ¿Por qué existe?

La «guerra» entre desarrolladores y administradores de sistemas lleva años entre nosotros. Es una guerra en silencio que quizá no hayáis visto, pero en la que no queréis estar 😉:

- **Desarrolladores**: «estos administradores… no me dan acceso root para instalar lo que quiero y no me dan más máquina y RAM»
- **Administradores**: «estos desarrolladores que no saben programar y tenemos que ir por detrás limpiando sus miserias…»

Es necesario para:

- Realizar desarrollos más ágiles.
- Despliegues automatizados.
- Ciclos de desarrollo más rápidos.
- Misma “imagen” en distintos servidores.
- Auto-escalado horizontal.
- ¡Porque mola! 😀

## ¿Quién lo usa?

Es la nueva moda, así que mucha gente lo está usando. Aparte:

- **Desarrolladores**: para desarrollar, para comprobar que su aplicación funciona en el entorno de test/pre-producción.
- **Administradores de sistemas**: para hacer despliegues en producción, probar nuevas versiones sin tener que actualizar el sistema operativo…

La frase «**es que en mi equipo funciona**» desaparece de la ecuación.

![one_more_time](/assets/images/one_more_time.jpg){: .w-50}

Un pase a producción supone:

- Antes:
  - Desarrolladores trabajan, hacen cambios, usan nuevas librerías, integran con servicios…
  - Les pasan el código a los administradores de sistemas.
  - Administradores de sistemas realizan el despliegue.
  - La aplicación parece funcionar.
  - La aplicación no funciona porque faltan las nuevas librerías, la integración no funciona…
  - Insultos varios.
  - Cliente enfadado.
  - Toca arreglar lo que no funciona.
  - Insultos varios.
  - Todo funciona.
  - Insultos varios.
- Ahora:
  - Desarrolladores trabajan, hacen cambios, usan nuevas librerías, integran con servicios…
  - Desarrolladores cambian la “receta” de despliegue para que coincida con los nuevos requisitos.
  - Administradores de sistemas realizan el despliegue en producción.
  - Todo funciona.
  - Nos vamos a tomar unas cañas.

## A tener en cuenta

La filosofía de Docker es que los contenedores tengan un ciclo de vida limitado, que no se pueda realizar modificaciones en ellos ya que **todo cambio debe de surgir en el momento de creación del contenedor.**

![thanos](/assets/images/thanos.png){: .w-25}

Vamos, que si no tenemos cuidado Thanos a nuestro lado es un parguela, porque **¡¡¡t****odo cambio a nivel de ficheros desaparecerá al borrar el contenedor!!!**, salvo que usemos volúmenes persistentes (luego hablamos de ello 😉).

## ¿Cómo lo hago funcionar?

La documentación oficial en [https://docs.docker.com/install/](https://docs.docker.com/install/)

Tenéis versiones para Windows y Mac también, pero los pasos en GNU/Linux, concretamente para Ubuntu, son:

    apt-get install apt-transport-https ca-certificates curl software-properties-common
    
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
    
    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu    $(lsb_release -cs) stable"
    
    apt-get update
    
    apt-get install docker-ce

Y ahora, ¿cómo ejecuto mi primer container docker?

    docker run --name mi-nginx -d -p 8080:80 nginx

    # entrar en el contenedor
    docker exec -it mi-nginx /bin/bash

Y ahora puedes ir al navegador y apuntar a **[http://IP:8080](http://IP:8080)**, donde IP es la del equipo donde está corriendo Docker.

Como estáis dentro del contenedor, modificad el fichero de la ruta **/usr/share/nginx/html/index.html** y después recargad el navegador.

![what](/assets/images/what.gif){: .w-50}

Lo que se ha hecho ha sido modificar un fichero dentro del container, pero **¡¡recuerda que este cambio desaparecerá cuando se borre el contenedor!!**

## Comandos (muy) básicos de Docker

- Ver los container Docker que tengo corriendo.
  - docker ps
- Ver todos los container Docker (incluso los parados).
  - docker ps -a
- Parar un container Docker.
  - docker stop CONTAINER\_ID
    - hay que pasarle el CONTAINER\_ID o el nombre.
- Levantar un container parado.
  - docker start CONTAINER\_ID
- Ver los logs de los últimos 5 minutos de un container Docker y ver los siguientes logs que vayan surgiendo.
  - docker logs CONTAINER\_ID -f –since 5m
- Borrar un container Docker **parado.**
  - docker rm CONTAINER\_ID
- Borrar todos los containers/volúmenes/imágenes que no se estén usando (a lo Thanos!).
  - docker system prune -a

## Pero, ¿cómo puedo hacer mis datos persistentes?

Como he dicho antes, cualquier cambio dentro del Docker desaparecerá al ser borrado, entonces, ¿cómo hago que eso no pase? Haciendo uso de lo que se denominan **“volúmenes”**.

La idea es «_mapear_» un directorio local de nuestro sistema anfitrión (el que corre Docker) en un directorio **dentro** del Docker. Como cuando montamos un USB en nuestro Linux y lo ponemos en una ruta.

Para ello:

    mkdir -p /opt/docker/mi-nginx

    docker run --name mi-nginx-persistente \\
      -v /opt/docker/mi-nginx:/usr/share/nginx/html -d -p 8081:80 nginx

¿Qué ha _pachado_?

![pachado](/assets/images/pachado.gif){: .w-50}

Hemos “montado” un path local (**/opt/docker/mi-nginx**) del host **anfitrión** dentro del container Docker en su ruta **/usr/share/nginx/html.** Ahora cualquier cambio que hagamos en esa ruta dentro del contenedor estará en el servidor y no desaparecerá al borrar el contenedor.

## Cómo funcionan por debajo los container Docker

De manera ultra resumida, funciona en base a “capas”. El ejemplo de la imagen de Nginx que hemos estado utilizando, podemos ver en [https://hub.docker.com/\_/nginx/](https://hub.docker.com/_/nginx/) las distintas opciones que hay, y cómo funciona por debajo en este fichero [Dockerfile](https://github.com/nginxinc/docker-nginx/blob/a22b9f46fe3a586b02d974f64441a4c07215dc5d/mainline/stretch/Dockerfile).

## ¿Qué más puedo instalar?

Cualquier cosa para la que ya exista una imagen creada: [https://hub.docker.com/](https://hub.docker.com/)

También nos podremos crear nosotros nuestra propia imagen a base de un Dockerfile [https://docs.docker.com/get-started/part2/#dockerfile](https://docs.docker.com/get-started/part2/#dockerfile)

¡Probad a crear otros containers!

## Y… ¿todo esto para qué?

Aparte de para aprender, ¿necesitas otra razón? 😉 Pues para hacer una infraestructura similar a esta:

![docker](/assets/images/docker_basico.png)

![scalated](/assets/images/scalated.gif){: .w-50}

En breve espero traeros un post más técnico que compense esta ida de olla 😉 Por cierto, en nuestro blog puedes encontrar otras entradas en las que hablamos de Docker.
