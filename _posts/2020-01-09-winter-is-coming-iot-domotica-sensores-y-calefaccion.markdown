---
layout: post
title:  "Winter is coming: IoT, domótica, sensores y calefacción"
date:    2020-01-09 00:00:00 +0200
tags: iot
---

Como si perteneciese a la casa Stark, permitidme decir: «Winter is coming» (si, lo sé, está muy repetido 😉 ). Y es que es verdad que el frío ha llegado, por lo que vamos a ver cómo controlar nuestra calefacción de casa a través de sensores y dispositivos conectados a nuestra red, todo ello a través de Internet y **haciendo uso de Software Libre**.

En Irontec llevamos años haciendo proyectos tecnológicos juntando sistemas punteros con sistemas electrónicos en busca de las mejores soluciones para nuestros clientes. Tenemos nuestro photocall automático, y sus proyectos hermanos megaselfie y happy selfie. Hemos hecho un espejo inteligente. Os hemos contado cómo hacer que las plantas nos llamen a través de Asterisk. Y hasta os hemos felicitado en dos ocasiones las Navidades encendiendo nuestro árbol y con un villancico hecho mediante motores paso a paso. Tenemos otros proyectos entre manos, con clientes punteros, que todavía no podemos desvelar, pero que ya os contaremos en nuestra sección de casos de éxito.

## Domótica e IoT

Wikipedia nos dice que la **[domótica](https://es.wikipedia.org/wiki/Dom%C3%B3tica)** es el sistema capaz de automatizar una vivienda aportando servicios de gestión energética, seguridad, bienestar y comunicación; mientras que el **[Internet de las cosas](https://es.wikipedia.org/wiki/Internet_de_las_cosas)** (o *IoT* en inglés) es un concepto que se refiere a la interconexión digital de objetos cotidianos con internet. Hoy en día, en lo que se refiere a automatizaciones caseras, y para no liarnos, se podría considerar que ambos conceptos van de la mano.

![esp01.jpg](/assets/images/esp01.jpg){: .w-50}
_ESP-01_

En los últimos años están apareciendo distintos sistemas (privativos) de empresas que nos venden sistemas domóticos, con su correspondiente controlador. En Irontec abogamos por el Software Libre y nos gusta más el **DIY** (*do it yourself*), sobre todo si con ello podemos aprender por el camino.

## ¿Qué necesitamos?

Para este proyecto vamos a necesitar distintos dispositivos electrónicos de coste muy bajo y la típica [Raspberry Pi](https://www.raspberrypi.org/) que compramos en su día, apenas hemos tocado, y se terminó quedando en el cajón 😀 .

La lista de la compra la compone:

- **Raspberry Pi**: Nos sirve cualquier modelo, pero cuanto más nuevo, más desahogada irá. Lo convertiremos en nuestro «gateway«, o sistema central de automatización gracias a [home-assistant](https://www.home-assistant.io/).
- [**ESP32**](https://en.wikipedia.org/wiki/ESP32) o [**ESP8266**](https://en.wikipedia.org/wiki/ESP8266): Son dos sistemas en chip (**SoC** o *System on a chip*) que tienen wifi y varios pins GPIO para poder recibir información de sensores o activarlos. Para este post voy a utilizar un ESP-01 como el de la foto de arriba, que sólo tiene 8 pins (alimentación, tierra, 2 GPIOs y serial).
  - Dependiendo del modelo que usemos lo podremos flashear mediante un cable USB o un conversor USB-Serial como el siguiente:

![usb-serial.jpg](/assets/images/usb-serial.jpg){: .w-50}
_Grabador USB-Serial_

- **Relé**: yo voy a usar una placa que se puede utilizar con los ESP8266-01 poniéndolo encima, y que funciona mediante un comando serie. Existen modelos más sencillos que funcionan activando una pata, pero elegí este por su tamaño.

![usb-serial.jpg](/assets/images/esp01-relay.jpg){: .w-50}
_ESP-Relay_

- **Alimentación** para todo esto. Dependiendo del modelo, se puede usar un conector de móvil o una mini batería.
- Paciencia y algo para picar/beber mientras trabajamos (que para eso, esta vez, es un proyecto casero y nos podemos permitir estar a varias cosas :D)

Lógicamente este hardware es el necesario para una instalación en la que la caldera se enciende a través de un enchufe o un relé, y por tanto será cambiar uno por otro. Tendríais que mirar cómo es vuestro modelo para tenerlo en cuenta para vuestro proyecto.

## Software necesario

El software que vamos a necesitar es el siguiente:

- **Home-Assistant**: Sistema de automatización casera con licencia Apache, que será el encargado de centralizar el estado de nuestros sensores y podremos acceder a ellos para activar/desactivar a través de un interfaz web. Si ya dispones de algún sensor comercial es posible que se pueda integrar con él, ya que Home-Assistant cuenta con más de 1500 integraciones actualmente. Como nota final, está escrito en Python.
- **ESPHome**: Nos va a permitir crear un firmware personalizado para poder configurar y controlar nuestra placa ESP32/ESP8266 simplemente configurando un fichero YAML. Gracias a ESPHome, Home-Assistant reconocerá nuestro SoC vía wifi y podremos controlarlo desde ese momento.

Para la programación de los ESP podríamos hacer uso del IDE de Arduino, y cargar las librerías necesarias, pero como iniciación creo que es mejor empezar con ESPHome. Es cierto que programando a bajo nivel con el IDE de Arduino podríamos llegar a controlar mucho mejor nuestro ESP, pero para ciertas tareas pequeñas, como el caso de hoy, me parece mejor hacer uso de integraciones sencillas.

## Instalando el software

Sabiendo todo lo anterior, vamos a ponernos a instalar y configurar el software que necesitamos:

### Raspberry Pi

No voy a entrar en detalle, ya que seguro que si tenéis una, habréis hecho muchas pruebas con ella. Lo mejor es que hagáis uso de Raspbian. Existe una imagen de Home-Assistant para Raspberry Pi, pero admito que no la he usado ya que he utilizado el método que luego os comento.

### Home-Assistant

Se puede realizar la instalación con Docker o la imagen que os comentaba antes, pero para hacer pruebas os aconsejo que lo hagáis con el entorno virtual de python. Para realizar la instalación de home-assistant:

    ruben@raspberrypi:~$  python3 -m venv home-assistant

    ruben@raspberrypi:~$ source home-assistant/bin/activate

    (home-assistant) ruben@raspberrypi:~$  pip3 install homeassistant
    Collecting homeassistant
    ...


    (home-assistant) ruben@raspberrypi:~$  hass

Y cuando se haya terminado de instalar, vamos a nuestro navegador y apuntamos al puerto 8123 de nuestra raspberry, http://IP_RASPBERRY:8123, donde nos realizará una serie de preguntas:

- Creación de usuario
- Nombre de la casa y ubicación

![Home Assistant](/assets/images/home-assistant.png){: .border}
_Home Assistant_

Y tras esto, vamos a empezar a configurar nuestros «cacharritos».

### ESPHome

Para la instalación de ESPHome os recomiendo hacerlo en vuestro equipo de trabajo, ya que vamos a estar flasheando el SoC y haciendo pruebas con él, por lo que hacerlo en la Raspberry puede resultar tedioso. De nuevo, os recomiendo hacer uso de los entornos virtuales de Python para la instalación:

    ruben@vega:~$  python3 -m venv esphome

    ruben@vega:~$ source esphome/bin/activate

    (hesphome) ruben@vega:~$  pip3 install esphome
    Collecting esphome
    ...


    (esphome) ruben@vega:~$  esphome config/ dashboard

Con el último comando lo que hacemos es levantar un servidor web en el puerto 6052 y lo que hagamos se va a guardar en el directorio «config». Vamos a nuestro navegador a **http://localhost:6052** y a partir de aquí nos aparecerá el Dashboard en el que podremos configurar nuestro ESP a través del asistente que aparece en la web.

## Programar el ESP e integración con Home Assistant

Tal como he comentado previamente voy a utilizar un ESP-01 (basado en ESP8266) y una placa con relé que se activa a través de un comando serie. Al entrar en el Dashboard de ESPHome nos aparece un enlace para añadir nuestro primer nodo ESP, cuyo asistente nos realizará las siguientes preguntas:

- **Nombre del nodo**: En nuestra casa domótica tendremos distintos nodos que realizarán distintas acciones, y con este nombre deberíamos saber qué es y qué hace. Para este caso le voy a llamar **caldera**.
- **Device type**:  Como existen distintos modelos de ESP32 y ESP8266 (cada uno con sus pins y características propias), ESPHome te deja elegir entre un buen listado de ellos. En este caso: **Espressif ESP-01 512k module**.
- **Configuración WiFi y OTA**: Nuestro dispositivo se puede conectar a nuestro WiFi y no sólo eso, ya que una vez puesto en marcha lo podremos actualizar vía OTA (lo que se llaman actualizaciones **Over-The-Air**). Para esta actualización será necesaria una contraseña.
Tras esto, nos aparecerá en el listado de nodos:

![Home Assistant](/assets/images/esphome_caldera.png){: .border}

Y si le damos a editar veremos que nos ha generado el siguiente código YAML:

    esphome:
      name: caldera
      platform: ESP8266
      board: esp01
    
    wifi:
      ssid: "wifi_casa"
      password: "contraseña_wifi"
    
      # Enable fallback hotspot (captive portal) in case wifi connection fails
      ap:
        ssid: "Caldera Fallback Hotspot"
        password: "f0m0fQPfRxB1"
    
    captive_portal:
    
    # Enable logging
    logger:
    
    # Enable Home Assistant API
    api:
      password: "ota_password"
    
    ota:
      password: "ota_password"

Como podéis ver, es un YAML y es bastante auto-explicativo. Hay dos apartados que nos ha generado que el configurador no nos ha preguntado:

- **AP**: Como pone el comentario previo, si el nodo no se consigue conectar al WiFi, automáticamente se pone en modo punto de acceso y nos podremos conectar a él.
- **API**: Por defecto, nos ha puesto la misma contraseña que para realizar la actualización OTA, pero podemos cambiarla. Esta contraseña será la que utilizará nuestro Home-Assistant para comunicarse con el nodo.

### Configurando el relé

Como he comentado, el relé se activa mediante comandos serial. Para ello, al YAML anterior he añadido la siguiente configuración (uart es la parte de configuración Serial):

    logger:
      baud_rate: 0
    
    uart:
      baud_rate: 115200 # speed
      tx_pin: GPIO1
      rx_pin: GPIO3
    
    
    switch:
      - platform: template
        name: 'boton_caldera'
        id: boton_caldera
        turn_on_action:
          - uart.write: [0xA0, 0x01, 0x01, 0xA2]
        turn_off_action:
          - uart.write: [0xA0, 0x01, 0x00, 0xA1]
        optimistic: true

Básicamente lo que estamos haciendo es crear un botón que se llama «boton_caldera», que cuando se activa (turn_on_action) manda un comando serial y cuando se apaga (turn_off_action) manda otro. Si el relé se activase a través de un pin, la configuración sería mucho más sencilla, y simplemente tendríamos que elegir el pin que se conecta al relé. Como estamos usando el serial, he desactivado el logger, ya que por defecto se hace uso del serial para ver qué hace el nodo (escanear las redes wifi, conectarse a la que le hemos puesto, configurar las acciones…).

## Conocer estado del nodo

Para asegurar que el nodo está conectado, podemos añadir la siguiente configuración al YAML. Al final, si terminamos teniendo muchos nodos y les ponemos batería, nos interesará saber cuándo se han quedado sin batería para cambiarla.

    binary_sensor:
      - platform: status
        name: "Nodo Caldera Status"

## Grabar Firmware

Como he comentado, el YAML de configuración es la manera sencilla de crear nuestro firmware personalizado, pero ahora falta grabarlo al ESP. Montamos el ESP al grabador USB-Serial, y conectado éste al ordenador, nuestro dashboard del ESPHome nos dirá que ha encontrado un nuevo puerto Serie. Lo elegimos y le damos al botón «Upload» para que grabe el firmware en el SoC. Con esto, lo que hace es:

- Se descarga las dependencias necesarias (para ello mira el YAML que hemos creado)
- Compila el firmware
- Lo graba en el ESP
- Reinicia el ESP

![OTA](/assets/images/esphome-serial-usb.png){: .border}
  
Tras grabarle el firmware, las siguientes actualizaciones las podemos hacer vía OTA, sin tener que enchufarlo al ordenador.

## Integración con Home Assistant

Una vez hayamos grabado el firmware, cogemos el ESP, lo conectamos al relé, lo alimentamos y veremos que automágicamente™ nuestro Home Assistant lo detectará apareciendo una notificación. Si vamos a ella, nos preguntará si queremos añadir el nuevo nodo «caldera» y nos pedirá la contraseña para poder conectarse a ella (recordad, la contraseña de la API que hemos puesto en el YAML). Una vez detectado, lo podemos añadir a un área de la casa (por defecto Home Assistant nos ha creado la Cocina, el Dormitorio y el Salón).

![Home Assistant esphome](/assets/images/nodo-caldera-funcionando.png){: .w-50}

Como se puede ver, en el interfaz de Home Assistant nos aparecerá el estado del nuevo nodo y un interruptor llamado «boton_caldera» (el nombre que le hemos dado en el YAML) que podremos activar y desactivar a nuestro antojo. Cuando lo hagamos veremos cómo el relé suena que se activa y desactiva, por lo que sabremos que está funcionando. Aparte, la placa que estoy usando también tiene un led para conocer el estado del relé, por lo que tenemos doble verificación.

Lo último que nos queda es conectar los cables de nuestra caldera a la boca correspondiente de la placa. Tendréis que comprobar con un tester cuál de las bocas es la opción «por defecto abierto», es decir, la que hace que no esté el circuito cerrado.

## Posibles mejoras

Podríamos hacer muchas mejoras a este sistema, pero la idea es que sea simple para genera un poco de hype 😉  Podríamos añadir otro nodo con un sensor de temperatura, hacer una integración automática entre la temperatura y el relé para que se encienda automáticamente, o incluso por horas…

Aparte, existe la aplicación de Home Assistant para móvil, por lo que si tenemos abierto el puerto 8123 y redirigido a la raspberry pi, podremos controlarlo estando fuera de casa, por lo que podemos encender la calefacción un rato antes de llegar a casa y así cuando lleguemos la casa esté caliente 😀

## Futuro de la IoT y domótica

Mientras escribía este post ha salido la noticia que cuenta que [Apple](https://www.apple.com/newsroom/2019/12/amazon-apple-google-and-the-zigbee-alliance-to-develop-connectivity-standard/), Google, Amazon y Zigbee (en la que participan Ikea y Samsung, entre otros) se alían para realizar un estándar abierto de conectividad de dispositivos inteligentes para el hogar. Este nuevo estándar basado en IP, y enfocado en la seguridad, pretende facilitar el desarrollo para los desarrolladores de dispositivos y que aumente la compatibilidad entre los dispositivos de distintas compañías.

El estándar será **open-source** y *royalty-free* por lo que cualquier compañía podrá utilizarlo, y esperemos que también facilite la expansión de hardware libre de bajo coste para poder tener nuestros sensores caseros. En la página web del proyecto [Connected Home](https://www.connectedhomeip.com/) tienen un poco más de información y dicen que esperan tener un borrador del estándar y una implementación open source para finales del 2020 (toca esperar).

Por su parte, Apple, para ir facilitando la creación de este estándar (o para posicionarse como base para el mismo, quién sabe) [ha decidido liberar partes de su Homekit Development Kit en GitHub](https://github.com/apple/HomeKitADK).

## Resumen

Como podéis ver, el configurar un relé y poder controlarlo no es difícil, por lo que puede ser un buen inicio de nuestra casa inteligente.

Además, con el nuevo estándar open source que se va a desarrollar, el futuro tiene mucha mejor pinta que el presente que tenemos ahora, en la que cada empresa ha estado haciendo un poco lo que quería.

Esperemos que os haya gustado el post de hoy y si tenéis algún proyecto tecnológico en el que entren en juego dispositivos electrónicos, IoT o cualquier otra tecnología puntera, no dudes en ponerte en contacto con nosotros, que estaremos encantados de colaborar.
