---
layout: post
title:  "OpenLDAP y passwords temporales (OTP)"
date:    2018-07-12 00:00:00 +0200
tags: openladp otp
---

Hoy traigo un post que tenía pendiente desde hace tiempo y que nos va a servir para poder tener OTP dentro de nuestro [OpenLDAP](https://www.openldap.org/) como sistema de autenticación.

## ¿Qué es OTP?

OTP viene de las siglas en inglés *One Time Password*, o contraseñas de un único uso. Estas contraseñas pueden generarse de dos maneras:

- **Sincronización de tiempo**: Tiene que haber sincronización de tiempo entre el servidor de autenticación y el cliente que genera la contraseña. Estas contraseñas son válidas durante un corto periodo de tiempo (normalmente 30 segundos).
- **Usando algoritmo matemático**:
  - Basadas en la contraseña anterior.
  - Basadas en un desafío (número aleatorio…).

Os sonará si tenéis algún servicio de internet con sistema de doble autenticación ([2FA](https://en.wikipedia.org/wiki/Multi-factor_authentication)) activado. No voy a dar más la chapa sobre esto, si queréis saber más, en la Wikipedia encontraréis mucha más información sobre [OTP](https://en.wikipedia.org/wiki/One-time_password), [TOTP](https://en.wikipedia.org/wiki/Time-based_One-time_Password_algorithm) y [HOTP](https://en.wikipedia.org/wiki/HMAC-based_One-time_Password_algorithm) 😉

## Instalación en OpenLDAP/slapd

Este post tiene como finalidad mostrar **cómo compilar e instalar el módulo OTP para OpenLDAP** en [Debian](https://www.debian.org/) 9 (Stretch). El problema que tenemos es que el módulo como tal ya existe en la rama master del [repositorio de OpenLDAP](https://github.com/openldap/openldap/tree/master/contrib/slapd-modules/passwd/totp),  pero hasta la versión 2.5 no saldrá de manera oficial, y casi todas las distribuciones tienen la 2.44 o 2.46. Así que vamos a ello:

![openldap](/assets/images/open_ldap_plain_caterpillar-999px.png){: .w-25}

### Preparación del entorno

Como ya he dicho, vamos a hacer la instalación en una Debian 9. Como vamos a tener que compilar el módulo, como preparación también vamos a tener que compilar el propio OpenLDAP. Para ello, necesitamos tener los repositorios «source» en nuestro fichero **/etc/apt/sources.list** :

    deb-src http://ftp.es.debian.org/debian/ stretch main contrib non-free

Y ahora actualizamos los repositorios y descargamos las dependencias para compilar OpenLDAP/slapd:

    apt-get update
    apt-get build-dep  slapd
    apt-get install dpkg-dev

Ahora como un usuario no privilegiado hacemos:

    mkdir debian
    cd debian
    apt-get source slapd

    cd openldap-2.4.44+dfsg/
    ./configure --prefix=/usr --libexecdir='${prefix}/lib' --sysconfdir=/etc --localstatedir=/var --mandir='${prefix}/share/man' --enable-debug --enable-dynamic --enable-syslog --enable-proctitle --enable-ipv6 --enable-local --enable-slapd --enable-dynacl --enable-aci --enable-cleartext --enable-crypt --disable-lmpasswd --enable-spasswd --enable-modules --enable-rewrite --enable-rlookups --enable-slapi --disable-slp --enable-wrappers --enable-backends=mod --disable-ndb --enable-overlays=mod --with-subdir=ldap --with-cyrus-sasl --with-threads --with-tls=gnutls --with-odbc=unixodbc

    make depend

Con esto hemos configurado las opciones como lo hace el propio mantenedor del paquete de Debian. Las opciones del *configure* las he sacado del fichero **debian/configure.options**. Podría parecer que ya está, pero no, todavía falta un poco más. Modificamos los siguientes ficheros:

- libraries/liblber/Makefile
- libraries/libldap/Makefile
- libraries/libldap_r/Makefile

Y donde pone «**VERSION\_OPTION = @VERSION\_OPTION@**» lo substituimos por «**VERSION\_OPTION = ./**«. Si no hacemos estos cambios obtendremos errores al compilar las librerías. Compilamos librerías y el propio OpenLDAP.

    cd libraries
    make
    cd ..
    make

**¡¡OJO!! No hagáis el mítico «make install»**

### Compilamos módulo TOTP

Como hemos comentado, el módulo TOTP está en fase beta y no se encuentra en Debian, por lo que necesitamos el código fuente del [repositorio](https://github.com/openldap/openldap/):

    cd

    git clone https://github.com/openldap/openldap.git openldap-git

El módulo se encuentra concretamente en el directorio **contrib/slapd-modules/passwd/totp/**. Copiamos este directorio dentro del directorio de los sources de debian y lo compilamos:

    cp -r openldap-git/contrib/slapd-modules/passwd/totp debian/openldap-2.4.44+dfsg/contrib/slapd-modules/passwd/

    cd debian/openldap-2.4.44+dfsg/contrib/slapd-modules/passwd/totp

    make

El módulo se compila y se queda en el directorio oculto **.libs** . Lo copiamos al directorio de OpenLDAP instalado por paquetes Debian. Lo bueno es que si tenemos varios servidores Debian 9 con OpenLDAP, podemos copiar esta librería en el resto de servidores sin tener que repetir el proceso.

    cp .libs/pw-totp.so\* /usr/lib/ldap/

## Configurar módulo TOTP

Desde las versiones 2.3.X OpenLDAP se configura en un árbol interno del ldap llamado **cn=config**. Vamos a explicar todos los pasos que hay que realizar:

### Activar el acceso a cn=config

Generamos la contraseña para el usuario que podrá conectarse al árbol de configuración, que será **cn=admin,cn=config** . Con el siguiente comando nos pedirá que introduzcamos la contraseña y nos devolverá el hash:

    slappasswd -h {SSHA}

    New password:
    Re-enter new password:
    {SSHA}Pz5LkigDQY1234QHW41uSXlPqxufraztn

Y generamos un fichero llamado **olcRootPW.ldif** con lo siguiente:

![openldap](/assets/images/acceso-config-OpenLDAP.png){: .w-50}

    dn: olcDatabase={0}config,cn=config
    changetype: modify
    add: olcRootPW
    olcRootPW: {SSHA}Pz5LkigDQY1234QHW41uSXlPqxufraztn

Y ahora tenemos que importarlo con:

    ldapmodify -Y EXTERNAL -H ldapi:/// -f ./olcRootPW.ldif

Ahora podemos hacer uso de un cliente de LDAP (como [jxplorer](http://jxplorer.org/)) para poder acceder a la sección de administración:

### Cargar módulo pw-totp

En la parte de configuración, con **jxplorer**, deberíamos tener **module{0}** .Dentro de este “nodo”, tenemos que seleccionar **olcModuleLoad**, dar “click derecho” y darle “add another value” que tiene que ser **{1}pw-totp**, por lo que nos debería quedar como en la imagen.

![openldap](/assets/images/modulo-añadido.png){: .w-50 .border}

### Añadimos overlay

Gerenamos el fichero **overlay.ldif** con el siguiente contenido:

    dn: olcOverlay={0}totp,olcDatabase={1}mdb,cn=config
    objectclass: olcOverlayConfig
    olcOverlay: {0}totp

Y añadimos la config haciendo:

    ldapadd -f  overlay.ldif  -D cn=admin,cn=config -x -W

## Configurar usuarios para autenticación TOTP

Ahora nos queda lo más importante, hacer que los usuarios tengan autenticación TOTP. Lo importante del usuario es lógicamente el campo **userPassword**. Lo tenemos que añadir como **texto plano** (desde jxplorer, phpldapadmin,…) y que sea algo tal que:

    {TOTP1}MNMGI3CZLBHGWZLONBVE22SFPJNFQSRQLJDVU3SZGNNGSQ3HHU6QU===

Hay que entender dos partes:

- **{TOTP1}**: Es para indicar que el usuario va a usar el nuevo módulo TOTP.
- **MNMGI3CZLBHGWZLONBVE22SFPJNFQSRQLJDVU3SZGNNGSQ3HHU6QU===**: es la base32 del base 64 de un string creado aleatorio. ¡**Este código tiene que ser diferente para cada usuario!**

Para generar el secreto que hay que darle al usuario:

    echo "qweasdzxc213ertdfgcvb" | base64 | base32
    MNMGI3CZLBHGWZLONBVE22SFPJNFQSRQLJDVU3SZGNNGSQ3HHU6QU===

## Autenticación TOTP

Vamos a comprobar que lo que hemos hecho hasta ahora funciona. Tras instalar el paquete **oathtool** generamos el TOTP:

    oathtool --totp -b MNMGI3CZLBHGWZLONBVE22SFPJNFQSRQLJDVU3SZGNNGSQ3HHU6QU===
    712909

Si queremos usar nuestro móvil con el programa [FreeOTP](https://freeotp.github.io/), necesitamos generar un código QR. Para ello necesitamos instalar el paquete **qrencode** y generar el código QR.

    echo -n "otpauth://totp/testldap:rgomez@irontec.com?    secret=MNMGI3CZLBHGWZLONBVE22SFPJNFQSRQLJDVU3SZGNNGSQ3HHU6QU&issuer=mycompany&period=30&digits=6&   algorithm=SHA1" | qrencode -s9 -o codigo_qr.png

El QR que genera lo tenéis aquí, y lo podéis escanear con FreeOTP. Los puntos a tener en cuenta en la URL del comando anterior son:

![openldap](/assets/images/codigo_qr.png){: .w-25}

- **otpauth://** : esquema/protocolo a utilizar
- **totp**: tipo de autenticación (podria ser totp o hotp).
- **testldap:rgomez@irontec.com**: es un string que indica la cuenta asociada y el nombre de la cuenta (normalmente un e-mail)
- **secret**: el secreto en base32. **Fijaros que hemos quitado los «=» del final del hash!!!**
- **issuer**: proveedor de asociado a esta cuenta
- **period**: cuanto va a durar la contraseña generada
- **digits**: número de dígitos que va a tener el TOTP
- **algorithm**: algoritmo de creación.

Un poco más de información en la web de [Google Authenticator](https://github.com/google/google-authenticator/wiki/Key-Uri-Format). Tanto el TOTP generado en FreeOTP como el de consola os tiene que dar los mismos dígitos 😀

Y confirmamos que la autenticación funciona ejecutando el siguiente comando y metiendo el código OTP que obtenemos del QR o el comando:

    ldapwhoami -D "cn=rgomez,ou=groups,dc=irontec,dc=com" -W -H ldap://localhost

## Conclusión

Aunque lo habitual es que OTP se use como autenticación de 2 factores, en este caso lo que hemos hecho ha sido que OpenLDAP lo use como sistema principal aunque no sea lo más óptimo. Esperemos que la versión 2.5 salga cuanto antes para que sea más sencillo de usar 😀  Y vosotros, ¿para qué usáis los OTP? ¿Los tenéis integrado en vuestros sistemas?
