---
layout: post
title:  "Creando un Docker Registry en nuestro GitLab"
date:    2019-11-05 00:00:00 +0200
tags: gitlab docker registry
---

Con el permiso de mi compañero [Jon](https://x.com/jon_latorre) y su post montando un docker registry «Como Dios Manda ™» , ahora que ya hicimos una [introducción, muy breve y desenfadada, a Docker]({% post_url 2018-10-29-introduccion-muy-breve-y-desenfadada-a-docker %}) y sabemos desarrollar con Docker vamos a retomar el asunto de crear un registry pero haciendo uso de nuestro entorno [GitLab]({% post_url 2015-06-09-gitlab-montando-un-entorno-git-multiusuario %}), para así unificar código y despliegue en una misma herramienta.

Han pasado casi 4 años y medio desde que os comentamos [cómo montar un entorno git multiusuario con GitLab]({% post_url 2015-06-09-gitlab-montando-un-entorno-git-multiusuario %}), y eso en informática es un mundo. Nosotros hemos estado actualizando nuestro GitLab de manera periódica para tener las nuevas características y las correcciones de seguridad, incluso hemos actualizado el sistema base a la última versión de Debian. Si tenéis alguna duda o necesitáis ayuda actualizando vuestro sistema GitLab no dudéis en poneros en contacto con nosotros.

## Repasando: ¿qué es un Registry en Docker y para qué lo quiero?

No está de más hacer un pequeño repaso de para qué sirve un registry y por qué queremos tener uno. Un registry es un **lugar donde almacenar imágenes de contenedores** y lo vamos a necesitar en cuanto queramos crear nuestras propias imágenes Docker.

![registry.png](/assets/images/registry.png){: .w-25}

La idea de tener un registry propio es el poder tenerlo cerca de nuestros entornos de producción, poder ahorrarnos ancho de banda y tiempo a la hora de realizar los despliegues. No hay que olvidar que por el camino vamos a aprender, y tal como dijo [Confucio](https://es.wikiquote.org/wiki/Confucio): «**Me lo contaron y lo olvidé; lo vi y lo entendí; lo hice y lo aprendí**«. Así que ¡¡manos a la obra!!

## ¿Cómo lo hago funcionar en GitLab?

Debemos tener al menos la versión 8.9 de GitLab para poder tener la opción de crear un registry. Es una versión bastante vieja, por lo que si lo has ido actualizando, no deberías tener problemas. Para activar la opción del registry añadiremos a nuestro fichero **/etc/gitlab/gitlab.rb** lo siguiente, permitiendo que todos los proyectos por defecto puedan hacer uso del Registry:

![gitlab-logo-gray-rgb](/assets/images/gitlab-logo-gray-rgb.png){: .w-50}

    gitlab_rails['registry_enabled'] = true

    gitlab_rails['gitlab_default_projects_features_container_registry'] = true

    gitlab_rails['registry_host'] = hublab.example.com

    registry['storage_delete_enabled'] = true

Tras esto tenemos que recargar la configuración de nuestro GitLab y debería empezar a funcionar

    gitlab-ctl reconfigure

Las imágenes creadas se almacenan en **/var/opt/gitlab/gitlab-rails/shared/registry**, por lo que si esperáis hacer uso intensivo y tener mucho histórico de imágenes, aseguraros que en ese path tenéis espacio suficiente.

## ¿Cómo lo puedo probar?

Vamos a crear un repositorio de pruebas que contenga los siguientes ficheros:

En el fichero **index.php** escribimos:

    <?php
    
    echo "<h1>HOLA MUNDO!</h1>";
    phpinfo();

No hay que explicar qué hace este index.php, ¿no? 😉

En el fichero **docker/Dockerfile**:

    FROM webdevops/php-nginx:7.4

    COPY . /app

    EXPOSE 80 443

Este Dockerfile es muy sencillo, básicamente lo que dice es que hace uso de la imagen de la gente de [webdevops](https://hub.docker.com/r/webdevops/php-nginx) y copia el contenido que tenemos en nuestro repositorio dentro del directorio **/app** de la imagen (el path que luego se usa en Nginx para servir la web). Esta imagen «padre» está preparada con un servicio web Nginx y la última _release candidate_ de PHP 7.4, así lo probamos 😀

En **.gitlab-ci.yml**:

    variables:
    REGISTRY_HOST: hublab.example.com


    build:
    stage: build
    image: docker:stable
    services:
        - docker:dind
    script:
        - docker login -u $CI_REGISTRY_USER -p $CI_BUILD_TOKEN $REGISTRY_HOST
        - docker build -t $REGISTRY_HOST/$CI_PROJECT_PATH:$CI_COMMIT_SHA -t $REGISTRY_HOST/$CI_PROJECT_PATH:latest -f docker/Dockerfile .
        - docker push $REGISTRY_HOST/$CI_PROJECT_PATH:$CI_COMMIT_SHA
        - docker push $REGISTRY_HOST/$CI_PROJECT_PATH:latest

Los ficheros **.gitlab-ci.yml** son los utilizados en GitLab para realizar la **Integración Continua** (_Continuous Integration_ en inglés), pero también para el desarrollo y el despliegue continuo (_Continuous Delivery_ y _Continuous Deployment_). En este caso lo vamos a utilizar para que en cada commit que realicemos en el repositorio, se cree una imagen nueva en el propio registry que hemos configurado. De manera muy resumida, y sin entrar en detalle, lo que hacemos en este fichero es:

- se conecta al registry
- crea la imagen teniendo en cuenta el fichero **docker/Dockerfile** que hemos añadido en el repositorio
- hace **_push_** de la imagen con el **_hash_** del commit
- hace **_push_** de la imagen y la marca como **latest**

No voy a negar que esto no es lo más óptimo, ya que lo más probable es que no nos interese hacer una imagen de cada _commit_, y lo ideal sería⁽™⁾ hacerlo sólo cuando creamos un tag. Pero eso lo dejo para que lo investiguéis vosotros 😉

### GitLab-Runner

Los [Gitlab-Runner](https://docs.gitlab.com/runner/register/) son los ejecutores de trabajos (_jobs_) que devuelven la información a GitLab. Estos _jobs_ pueden realizar cualquier cosa, desde analizar que el código es óptimo y no tiene problemas de ejecución, hasta, como es el caso de hoy, realizar una imagen y subirla a nuestro registry. Nosotros hemos ido generando una red de _runners_ por nuestra infraestructura para realizar distintas tareas, pero para la de hoy necesitamos uno (nosotros lo tenemos instalado en el mismo servidor que Gitlab) que tenga la siguiente configuración:

    [[runners]]
    name = "gitlab.example.com"
    url = "https://gitlab.example.com"
    token = "TOKEN_RUNNER"
    executor = "docker"
    [runners.custom_build_dir]
    [runners.docker]
        dns = ["8.8.8.8"]
        tls_verify = false
        image = "docker:latest"
        privileged = true
        disable_entrypoint_overwrite = false
        oom_kill_disable = false
        disable_cache = false
        volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
        shm_size = 0
    [runners.cache]
        [runners.cache.s3]
        [runners.cache.gcs]

Tras esto, cada commit que hagamos, se ejecutará un pipeline, que será ejecutado por nuestro runner, que generará una imagen nueva. En la siguiente imagen podemos ver cómo en mi proyecto de pruebas se han generado varias imágenes Docker

![registry.png](/assets/images/imagenes_registry.png){: .border}

## Probando nuestra nueva imagen Docker

Ahora que ya tenemos nuestra imagen Docker, vamos a asegurar que podemos descargar la imagen y la podemos ejecutar en nuestro entorno Docker. Primero nos tenemos que loguear contra nuestro nuevo registry:

    docker login hublab.example.com       

    Username: rubengomez
    Password: 

    WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
    Configure a credential helper to remove this warning. See
    https://docs.docker.com/engine/reference/commandline/login/#credentials-store

    Login Succeeded

El registry está autenticado con los credenciales de GitLab, y nos avisa que se guardará la contraseña en nuestro **~/.docker/config.json**, así que cuidado. Y ahora levantamos un contenedor con la imagen generada:

    docker run --name mi-prueba -p 8080:80 -d hublab.example.com/rubengomez/prueba:latest

Lo que hacemos es mapear nuestro puerto 8080 al puerto 80 del contenedor, vamos a a nuestro navegador al puerto 8080 y vemos el contenido de nuestro **index.php** que está en nuestra imagen:

![php_registry.png](/assets/images/php_registry.png){: .border}

## Resumen

En el post de hoy hemos hecho que nuestro servicio GitLab, aparte de servir como repositorio de código fuente, nos sirva a modo de Registry para nuestra infraestructura Docker. No hemos entrado en detalle, pero también hemos realizado nuestro primer _job_ de integración continua, que genera una imagen de nuestro repositorio. Y por último, hemos desplegado la nueva imagen. En breve os traeremos nuevos posts en los que  profundizaremos más en Docker, en integración continua, despliegue continuo y análisis de código.

Esperamos que os haya gustado y os sirva como base para vuestros despliegues. Ya sabéis que si necesitáis cualquier ayuda estamos aquí para poder ayudaros en lo que necesitéis.
