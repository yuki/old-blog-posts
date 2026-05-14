---
layout: post
title:  "Historia (informal) de los sistemas de control de versiones"
date:    2015-05-26 00:00:00 +0200
tags: git svn
---


Diría que casi cualquier persona que ha usado alguna vez un ordenador ha utilizado un sistema de control de versiones, aunque probablemente no se haya dado cuenta de ello. Pero, **¿qué es exactamente un sistema de control de versiones?**

Como su propio nombre indica, es un sistema que nos **ayuda con la gestión de los cambios que se realizan sobre un fichero a lo largo del tiempo**. Una **versión** es el estado de dicho fichero en un momento concreto en la vida del mismo, y podremos ver los cambios realizados en él antes y después de dicho momento, hasta llegar a la última versión o estado actual.

En resumen, nos ayuda a que, si hemos realizado una modificación en un fichero, podamos volver al estado anterior, siempre y cuando este fichero, y su estado anterior, estuviese «versionado» en el sistema. Por lo tanto, hay que tener en cuenta que siempre que queramos guardar una versión del fichero **tendremos que acordarnos de realizar un «commit»** (palabra técnica que significa que enviamos el estado al sistema de control de versiones). No sirve de nada realizar modificaciones si no realizamos este último paso.

![not_a_programmer.jpg](/assets/images/not_a_programmer.jpg){: .w-15 }

Las modificaciones realizadas en los ficheros son enviadas a un **repositorio** donde serán almacenadas, donde se realiza la «magia» de mantener cada una de estas versiones y dónde se puede consultar el histórico de las mismas.

En el caso de los programadores, el poder realizar este versionado del código nos ayuda a ver **cuál es la evolución del mismo** y poder ir atrás, a un estado en el que el código era funcional, o poder observar cómo era el proyecto en sus primeras fases.

Y como punto extra, nos ayuda a realizar la **unificación del código fuente** cuando se realiza un trabajo colaborativo entre varios desarrolladores.

La **evolución de los sistemas de control de versiones** se podría resumir de la siguiente manera:

## Copias añadiendo fechas

Probablemente el más usado por todo el mundo, gente técnica incluida. Quién no ha realizado un trabajo de clase en el que ha terminado con varias versiones, tal que:

    trabajo.odt
    trabajo-2015-05-08.odt
    trabajo-2015-05-07.odt

A medida que iba aumentando el proceso de escritura y que los nervios se apoderaban de nosotros por la posibilidad de perder algo de información, hasta llegábamos a poner la hora!!!:

    trabajo-2015-05-11_10-20.odt
    trabajo-2015-05-11_10-45.odt
    trabajo-2015-05-11_11-30.odt

Pero ya lo mejor era cuando la información estaba escrita y sólo nos quedaba realizar la maquetación antes de imprimir:

    trabajo-final.odt
    trabajo-final2.odt
    trabajo-final-de-verdad.odt
    trabajo-final-ultimisima-version-a-presentar.odt

Parecía como los juegos de mesa, en cada casa había reglas propias.

## CVS

[CVS](http://savannah.nongnu.org/projects/cvs) fue el primer sistema de control de versiones real utilizado por muchos de nosotros, y en mi caso, la sensación fue «¿por qué no lo habré conocido antes?».

Con el nombre de «Concurrent Versions System» nos sentíamos como que por fin estábamos usando algo profesional, y lo mejor es que en él veíamos cosas que nos sonaban del proceso anterior. Cada fichero contaba con su propia versión que era un número y, por lo tanto, sabíamos que la versión 6 del fichero era posterior a la versión 5 del mismo.

Aún así, siempre daba la sensación de que había procesos que se hacían con miedo por relatos escuchados a «los mayores» acerca de rotura de repositorios al realizar un commit «complicado». Por lo tanto, algunos nos curábamos en salud haciendo una **copia local,** por si acaso, antes de hacer algún cambio, no vaya a ser que hubiese algún desastre.

## Subversion

No me peleé mucho con CVS porque por aquel entonces [Subversion](https://subversion.apache.org/) ya estaba de moda. Nació con el fin de terminar con el reinado de CVS mejorando lo que éste hacía, corrigiendo bugs y añadiendo nuevas características que se veían necesarias por aquel entonces.

![svn files](/assets/images/svn-files.png)

**La creación de repositorios parecía más fácil**. Existía una **versión global** a nivel de repositorio, lo que te podía dar una idea de hacía cuánto se había creado un fichero. La creación de **ramas y tags** era algo que yo descubría por primera vez y que, aún siendo complejo, se podía ver el potencial de esas características… Vamos, que parecía un nuevo mundo por descubrir aquel entonces.

Había cosas que molestaban, como eso de que cuando querías hacer un commit y no podías, él mismo no hiciese un update del repositorio local o te dijese qué querías hacer. O ese desperdicio de crear tanto directorio «.svn» que terminaron por corregir en versiones posteriores.

``` terminal
    svn ci -m'#0048735 Add perfdata to check_swapping' check_swapping
```

Aún a pesar de sus fallos, Subversion es uno de los paquetes de **software libre** más utilizado y que se seguirá utilizando durante mucho tiempo, ya que sigue evolucionando en cada nueva versión.

## GIT

Parece mentira que hace poco más de un mes se [cumplieran 10 años](http://www.linux.com/news/featured-blogs/185-jennifer-cloer/821541-10-years-of-git-an-interview-with-git-creator-linus-torvalds) del anuncio por parte de **Linus Torvalds** de la creación de **GIT**.

![git log](/assets/images/kernel.png)

Pero creo que cuando más gente le quisimos echar un ojo fue allá por el 2007, cuando [Linus dio una charla en las oficinas de Google](https://www.youtube.com/watch?v=4XpnKHJAok8) donde mostraba su manera de ser y ponía del revés a Subversion, y sobre todo a CVS, en los primeros cinco minutos. Creo que el querer probarlo era más por el morbo de haber visto la charla que por las razones técnicas. Conociendo los antecedentes de Linus, se preveía que algo grande había en GIT.

{% include embed/youtube.html id='4XpnKHJAok8' %}

Y así fue. Muchos de nosotros descubrimos qué era eso de utilizar un **sistema de control de versiones distribuido** y cómo el no depender de un repositorio central te quitaba la presión de tener que estar todo el rato actualizando tu código con lo hecho por el resto de compañeros. La **facilidad de crear ramas** de desarrollo te dejaba alternar entre **escribir _features_ nuevas o corregir _bugfixes_ en cuestión de segundos**, sin tener que depender, nuevamente, de la lentitud de un repositorio central.

Cómo hacer un simple _«git log»_ te mostraba la gran diferencia de velocidad con Subversion, y aunque no conocías todo lo que podías hacer con GIT (y seguías usándolo casi del mismo modo que usabas SVN), sabías que había merecido la pena el cambio. Poco a poco ibas descubriendo opciones nuevas como **juntar varios commits**, hacer **cherry-picks** de otras ramas, utilizar **stash** antes de cambiar de rama…

Y aunque no pudieses cambiar el repositorio central de Subversion de la empresa, tenías **git-svn** para utilizarlo de manera local y así poder seguir aprendiendo e ir dejando SVN de lado.

``` bash
ruben@host:/tmp/$ git svn clone -Ttrunk -ttags -bbranches http://dev.irontec.com/svn/configs
```

A medida que iban apareciendo versiones nuevas y se iba generalizando su uso, también fueron surgiendo proyectos para dar almacenaje a repositorios como [Github](https://github.com/yuki) y, lo más importante, **proyectos de software libre para tener nuestros propios repositorios con autenticación y buen aspecto como [Gitlab](https://gitlab.com/gitlab-org/gitlab-ce/tree/master#README)**.

Pero una de las mejores opciones que te da GIT es la opción de no obligarte a seguir unas reglas como hasta ese momento se tenía, si no a poder crear un **sistema propio de trabajo** (o aprender de [cómo trabajan otros con git](http://nvie.com/posts/a-successful-git-branching-model/)).

En otro artículo explicaremos cómo montar nuestro sistema GIT centralizado con autenticación utilizando Gitlab y así no tener más excusas para no **migrar a GIT**, basándonos en las experiencias de nuestro día a día en el departamento de Sistemas.
