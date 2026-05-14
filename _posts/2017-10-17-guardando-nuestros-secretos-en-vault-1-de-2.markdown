---
layout: post
title:  "Guardando nuestros secretos en Vault (1 de 2)"
date:    2017-10-17 00:00:00 +0200
tags: vault passwords
---

Todos tenemos secretos que guardar. Hay gente que los mantiene en la memoria, otros prefieren escribir un diario donde volcar ahí todos sus pensamientos y otros simplemente prefieren olvidar. El mundo de la informática no iba a ser menos, pero **a los secretos los llamamos contraseñas**. Ahora mismo puedes estar pensando: _«las contraseñas no se pueden olvidar!!!»_, pero veremos cómo esto no es del todo cierto.

Hoy vamos a hacer una breve introducción a **[Vault](https://www.vaultproject.io/)**. **Un sistema de almacenamiento de contraseñas/secretos que nos está gustando mucho últimamente.** Veremos cómo instalarlo, para qué sirve y cómo podemos utilizarlo desde la **API** que tiene integrada.

## ¿Qué es Vault?

Ya hemos adelantado que Vault es un **sistema de almacenamiento y acceso a secretos.** ¿Y qué es un secreto? Un secreto es cualquier cosa que quieres t**ener bajo un control estricto de acceso**, como pueden ser claves de acceso a APIs, contraseñas, certificados…

![vault-logo](/assets/images/vault-logo.png){: .w-25}

Vault permite un interfaz unificado a cualquier secreto a la par que provee un sistema de control de acceso a los mismos, y un sistema de log de acceso. Para entender un poco cómo funciona, hay que entender las **distintas partes** que tiene:

- **Sistema de almacenamiento seguro:** Vault permite almacenar los secretos clave/valor en distintos sistemas de almacenamiento: disco duro, base de datos, consul… Lo importante es que antes de que el secreto sea almacenado se cifra.
- **Secretos dinámicos:** Podemos crear secretos de manera dinámica para ciertos sistemas. Por ejemplo, si necesitamos acceder a una base de datos, Vault nos podría generar unos credenciales de acceso con los permisos necesarios que luego usaríamos. Una vez generado este secreto, Vault se encargará automáticamente de revocarlos pasados el tiempo de expiración configurado.
- **Cifrado de datos al vuelo:** Vault permite el cifrado y descifrado de datos sin necesidad de almacenarlo.
- **Expiración y renovación:** Todos los secretos tienen un tiempo de expiración, tras el cual, Vault revoca los secretos. Lo bueno es que el cliente que pide el secreto es capaz de realizar una renovación del tiempo de expiración mediante la API.
- **Revocación:** Como ya hemos comentado, Vault automáticamente revoca los secretos pasados el tiempo de expiración correspondiente.

Con esto nos podemos hacer una idea de los usos que le podemos dar. Desde un sistema de **almacenamiento de contraseñas**, hasta un sistema de **creación de credenciales temporales** para acceder a un servicio externo como un MySQL o SSH. **La ventaja que tiene es que Vault permite la extensión mediante plugins**, por lo que es probable que el uso que se le pueda dar se incremente en próximas versiones.

### ¿Cómo funciona Vault?

No hay nada como ver un dibujo para intentar entender cómo funciona Vault:

![vault-logo](/assets/images/vault-arquitectura.png)

Tal como se puede ver, existe una clara separación de los componentes que están dentro y fuera de la barrera (barrier) de seguridad de Vault. Cuando el servicio arranca, a través de la configuración, le tenemos que **indicar un sistema de almacenamiento**, y tal como hemos dicho previamente, antes de guardar el secreto ya **estará cifrado.** Esto es muy importante para asegurar que la cadena de seguridad no se rompe.

Por otro lado, **cualquier petición** que se le realiza a Vault **es mediante una API**. No tendría sentido hacer peticiones por HTTP plano, ya que una vez el secreto sale de Vault ya no estará cifrado, por lo que en el siguiente post veremos cómo securizar Vault para ello. Antes de realizar cualquier petición nos deberemos haber autenticado con cualquiera de los sistemas permitidos (token, LDAP, github…)

Por último, hay que entender que **cuando arrancamos Vault, el sistema está sellado/precintado**. ¿Y esto qué quiere decir? Que hasta que no se haya realizado el desellado del sistema, no se podrá realizar ninguna petición y por tanto no se podrá acceder a ningún secreto guardado previamente.

![vault keys](/assets/images/vault-shamir-secret-sharing.png){: .w-50}

Cuando se inicia Vault por primera vez se **genera una clave de cifrado** que es la que se utiliza **para proteger toda la información**. Esta clave está protegida por una clave maestra, que por defecto se «parte» en 5 claves compartidas mediante el [algoritmo de compartición de Shamir](https://es.wikipedia.org/wiki/Esquema_de_Shamir). Podremos generar la clave maestra con 3 de estas 5 claves compartidas. El número de claves compartidas y el número necesarios de ellas para generar la clave maestra es configurable. Con éstas, se podrá desellar Vault y ya podremos acceder a los secretos, pero antes de eso, el cliente se tendrá que autenticar para asegurar que tiene acceso a los secretos.

Para entender este sistema podríamos utilizar el símil de las películas cuando se está a punto de una [guerra termonuclear mundial](http://www.imdb.com/title/tt0086567/). Para lanzar los misiles suele ser habitual que al menos haya dos llaves que custodian sendos soldados, las cuales son introducidas y giradas a la vez para poder activar el sistema. La introducción y girar las llaves sería como meter las «shared keys», con ello **desprecintamos** Vault. Luego, para lanzar el misil hay veces que hay que meter un código, que eso sería el sistema de **autenticación**. Y por último, pulsamos el botón rojo para lanzar el misil, que sería la **petición** del secreto.

## Instalando Vault

Actualmente **no existen paquetes oficiales para las distintas distribuciones,** y parece que no tienen intención de hacerlos, pero vamos a ver cómo la instalación es muy sencilla. Existe la posibilidad de compilar el código fuente, pero hoy vamos a ir a lo sencillo.

La instalación básicamente se basa en **descargar el código** compilado desde la web de [descargas de Vault](https://www.vaultproject.io/downloads.html), y descomprimir el el fichero zip. Después, lo movemos a **/usr/local/sbin** para dejarlo dentro del PATH.

    wget https://releases.hashicorp.com/vault/0.8.3/vault_0.8.3_linux_amd64.zip

    unzip vault_0.8.3_linux_amd64.zip

    mv vault /usr/local/sbin

Vault funciona en modo cliente servidor, y para ambos se utiliza el mismo binario **vault**.

### Arrancando Vault en modo desarrollo

El post de hoy es una introducción básica a Vault, así que vamos a arrancarlo **en modo desarrollo.** Para hacer nuestras pruebas de hoy es suficiente, y en el próximo post veremos cómo hacer una configuración permanente del servicio.

Hay que entender que **el modo desarrollo de Vault es únicamente para hacer pruebas**. Las características del modo desarrollo son:

- El servidor está inicializado y desellado.
- Almacenamiento en memoria. **Si se para el servicio se pierde todo.**
- Acceso a través de localhost y sin certificado TLS. Todo el acceso es mediante HTTP plano
- Autenticación automática. La contraseña de root es almacenada directamente
- Únicamente hay una clave de desprecintado (no 5 como hemos visto previamente).

Teniendo en cuenta estas características, vemos cómo el sistema ya no es tan seguro, pero ese es el fin del modo desarrollo, hacer pruebas en un entorno volátil.

    root@vault:~# vault server -dev
    ==> Vault server configuration:

                        Cgo: disabled
            Cluster Address: https://127.0.0.1:8201
                Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", tls: "disabled")
                Log Level: info
                    Mlock: supported: true, enabled: false
            Redirect Address: http://127.0.0.1:8200
                    Storage: inmem
                    Version: Vault v0.8.3
                Version Sha: 6b29fb2b7f70ed538ee2b3c057335d706b6d4e36

    ==> WARNING: Dev mode is enabled!

    In this mode, Vault is completely in-memory and unsealed.
    Vault is configured to only have a single unseal key. The root
    token has already been authenticated with the CLI, so you can
    immediately begin using the Vault CLI.

    The only step you need to take is to set the following
    environment variables:

        export VAULT\_ADDR='http://127.0.0.1:8200'

    The unseal key and root token are reproduced below in case you
    want to seal/unseal the Vault or play with authentication.

    Unseal Key: 89j3j/OqDkCFgYpPsXXeIPCsjpZRiIaH3qa1ECIr/hw=
    Root Token: 01f0c096-b7fd-513d-fdec-953b830deb15

    ==> Vault server started! Log data will stream in below:

He destacado las líneas más importantes. En la primera, arrancamos el servicio con el comando «vault server -dev», tan fácil como eso. En la salida del comando vemos cómo nos indica que ha levantado un _«listener»_, y que **TLS está desactivado**. Nos indica que para acceder la dirección es **[http://localhost:8200](http://localhost:8200),** y nos recuerda que estamos en modo «Dev» (desarrollo).

Por último, nos indica que tenemos que exportar una variable de entorno para acceder a Vault, y nos aparece la clave de desellado (que no vamos a necesitar, salvo que sellemos Vault) y el token del usuario root, que necesitaremos para acceder a través de la API.

### Crear y consultar secretos

**Ya sabemos qué hace Vault, lo tenemos instalado y arrancado, así que ahora toca jugar un poco con él.**

Como ya hemos dicho, en el modo desarrollo el sistema está desprecintado, tal como nos muestra el siguiente comando, si no, tendríamos que meter la clave de desellado:

    root@vault:/opt# vault unseal
    Vault is already unsealed.

Vamos a ver los «backends» que están montados por defecto:

    root@vault:/opt# vault mounts
    Path        Type       Accessor            Plugin  Default TTL  Max TTL  Force No Cache  Replication Behavior  Description
    cubbyhole/  cubbyhole  cubbyhole_861c7a67  n/a     n/a          n/a      false           local                 per-token private secret storage
    secret/     kv         kv_8ce5f2c4         n/a     system       system   false           replicated            key/value secret storage
    sys/        system     system_28c35f87     n/a     n/a          n/a      false           replicated            system endpoints used for control, policy and debugging

He resaltado el backend que vamos a utilizar, que es «secret/», que si miramos nos indica que es «**key/value secret storage**«. Es decir, «secret» es el sistema clave-valor donde se van a almacenar los secretos que vamos a crear a continuación.

**Vamos a crear el secreto** «secreto1», y después vamos a listar los secretos que tenemos en el backend «secret» y por último leerlo:

    root@vault:/opt# vault write secret/secreto1 hola="que tal"
    Success! Data written to: secret/secreto1

    root@vault:/opt# vault list secret/
    Keys
    ----
    secreto1


    root@vault:/opt# vault read secret/secreto1
    Key                     Value
    ---                     -----
    refresh\_interval        768h0m0s
    hola                    que tal

Tal como se puede ver, con el CLI de vault los parámetros son muy intuitivos. De momento vamos a dejar la clave «**refresh\_interval**» para el siguiente post, pero como nos podemos imaginar, por el nombre, es el tiempo de expiración que podemos ponerle a un secreto.

Vamos a hacer uso de la API con el comando **curl** para ver lo fácil que es realizar los mismos pasos. Dado que la API está fuera de la barrera del core, tenemos que estar autenticados, de ahí que pasemos el token de root como parámetro al comando:

    root@vault:/opt# curl -D - --header "X-Vault-Token: 01f0c096-b7fd-513d-fdec-953b830deb15" --request POST --data '{"hola2":"que tal"}' http://localhost:8200/v1/secret/secreto2
    HTTP/1.1 204 No Content
    Cache-Control: no-store
    Content-Type: application/json
    Date: Mon, 16 Oct 2017 07:36:49 GMT


    root@vault:/opt# curl --header "X-Vault-Token: 01f0c096-b7fd-513d-fdec-953b830deb15" --request LIST  http://localhost:8200/v1/secret/ -s | jq .
    {
    "request\_id": "73cf3aa9-2b41-7809-74a8-f489271b8d61",
    "lease\_id": "",
    "renewable": false,
    "lease\_duration": 0,
    "data": {
        "keys": \[
        "secreto1",
        "secreto2"
        \]
    },
    "wrap\_info": null,
    "warnings": null,
    "auth": null
    }


    root@vault:/opt# curl --header "X-Vault-Token: 01f0c096-b7fd-513d-fdec-953b830deb15" --request GET  http://localhost:8200/v1/secret/secreto2 -s  | jq .
    {
    "request\_id": "3f8710e6-5fb8-a887-e413-4b4e6c2eb3f6",
    "lease\_id": "",
    "renewable": false,
    "lease\_duration": 2764800,
    "data": {
        "hola2": "que tal"
    },
    "wrap\_info": null,
    "warnings": null,
    "auth": null
    }

Cómo podéis ver el primer comando curl es para crear un secreto nuevo llamado «secreto2», al cual le paso una cabecera **X-Vault-Token** indicando el token de root que nos ha dado Vault al arrancar en modo desarrollo. El tipo de petición es **POST** y le paso un json con los datos clave-valor del secreto. La respuesta que nos da Vault es un 204 indicando que todo ha ido bien.

La segunda petición es parecida a la primera, sólo que no pasamos datos y es de tipo **LIST**, para listar los secretos que tenemos. La salida de curl la he pasado al comando **jq** simplemente para que el json que nos devuelve la petición se vea mejor. Si os fijáis en la salida, lo que interesa dentro del json es el campo «**data**«, y dentro nos da un array llamado **keys** con los nombres de los secretos que tenemos actualmente: secreto1 y secreto2.

El tercer curl es para obtener el valor del secreto recién creado «secreto2». La diferencia con el anterior es que la petición es **GET**. Fijaros que dependiendo de la petición que hagamos, la URL a la que preguntamos varía poco, pero al final es como cualquier API. Un poco más de información en la [documentación oficial de Vault](https://www.vaultproject.io/api/secret/kv/index.html).

## Resumen

Hemos visto **cómo funciona Vault de manera interna** y **cómo podemos hacer uso del CLI y de la API para poder crear y ver los secretos creados.** En el próximo post haremos una **configuración real** del sistema para poder ponerlo en producción, ya que como habéis visto hoy, el sistema lo hemos probado en modo desarrollo y no existe persistencia de datos.

Espero que os haya gustado el post y empecéis a guardar vuestros secretos de manera segura 😉 Para cualquier duda, dejadnos un comentario 😀
