# Ejercicio 5: Control de acceso, autentificación y autorización

## Control de acceso


El Control de acceso en un servidor web nos permite determinar desde donde 
podemos acceder a los recursos del servidor.

En apache2.2 se utilizan las siguientes directivas: order, allow y deny. Un 
buen manual para que quede más claro lo puedes encontrar en este enlace. La 
directiva satisfy controla como el se debe comportar el servidor cuando 
tenemos autorizaciones de control de acceso (allow, deny,…) y tenemos 
autorizaciones de usuarios (require).

En apache2.4 se utilizan las siguientes directivas: Require, RequireAll, 
RequireAny y RequireNone

**1.** Comprueba el control de acceso por defecto que tiene el virtual host 
por defecto (000-default).

El virtualHost por defecto, utiliza Require seguido de 


## Autentificación básica

El servidor web Apache puede acompañarse de distintos módulos para proporcionar 
diferentes modelos de autenticación. La primera forma que veremos es la más 
simple. Usamos para ello el módulo de autenticación básica que viene instalada 
“de serie” con cualquier Apache: mod_auth_basic. La configuración que tenemos 
que añadir en el fichero de definición del Virtual Host a proteger podría ser 
algo así:

```
  <Directory "/var/www/miweb/privado">
    AuthUserFile "/etc/apache2/claves/passwd.txt"
    AuthName "Palabra de paso"
    AuthType Basic
    Require valid-user
  </Directory>
```

El método de autentificación básica se indica en la directiva AuthType.

* En Directory escribimos el directorio a proteger, que puede ser el raíz de 
nuestro Virtual Host o un directorio interior a este.

* En AuthUserFile ponemos el fichero que guardará la información de usuarios y 
contraseñas que debería de estar, como en este ejemplo, en un directorio que 
no sea visitable desde nuestro Apache. Ahora comentaremos la forma de generarlo.

* Por último, en AuthName personalizamos el mensaje que aparecerá en la 
ventana del navegador que nos pedirá la contraseña.

* Para controlar el control de acceso, es decir, que usuarios tienen permiso 
para obtener el recurso utilizamos las siguientes directivas: AuthGroupFile, 
Require user, Require group.

El fichero de contraseñas se genera mediante la utilidad htpasswd. Su sintaxis 
es bien sencilla. Para añadir un nuevo usuario al fichero operamos así:

```
$ htpasswd /etc/apache2/claves/passwd.txt carolina
New password:
Re-type new password:
Adding password for user carolina
```

Para crear el fichero de contraseñas con la introducción del primer usuario 
tenemos que añadir la opción -c (create) al comando anterior. Si por error la 
seguimos usando al incorporar nuevos usuarios borraremos todos los anteriores, 
así que cuidado con esto. Las contraseñas, como podemos ver a continuación, 
no se guardan en claro. Lo que se almacena es el resultado de aplicar una 
función hash:

```
josemaria:rOUetcAKYaliE
carolina:hmO6V4bM8KLdw
alberto:9RjyKKYK.xyhk
```

Para denegar el acceso a algún usuario basta con que borremos la línea 
correspondiente al mismo. No es necesario que le pidamos a Apache que vuelva 
a leer su configuración cada vez que hagamos algún cambio en este fichero 
de contraseñas.

La principal ventaja de este método es su sencillez. Sus inconvenientes: lo 
incómodo de delegar la generación de nuevos usuarios en alguien que no sea un 
administrador de sistemas o de hacer un front-end para que sea el propio 
usuario quien cambie su contraseña. Y, por supuesto, que dichas contraseñas 
viajan en claro a través de la red. Si queremos evitar esto último podemos 
crear una instancia Apache con SSL.

**Cómo funciona este método de autentificación**

Cuando desde el cliente intentamos acceder a una URL que esta controlada por 
el método de autentificación básico:

1. El servidor manda una respuesta del tipo 401 HTTP/1.1 401 Authorization 
Required con una cabecera WWW-Authenticate al cliente de la forma:

    > WWW-Authenticate: Basic realm="Palabra de paso"

2. El navegador del cliente muestra una ventana emergente preguntando por el 
nombre de usuario y contraseña y cuando se rellena se manda una petición 
con una cabecera Authorization

    > Authorization: Basic am9zZTpqb3Nl

En realidad la información que se manda es el nombre de usuario y la 
contraseña en base 64, que se puede decodificar fácilmente con cualquier 
utilidad.

## Autentificación tipo digest

La autentificación tipo digest soluciona el problema de la transferencia de 
contraseñas en claro sin necesidad de usar SSL. El procedimiento, como veréis, 
es muy similar al tipo básico pero cambiando algunas de las directivas y 
usando la utilidad htdigest en lugar de htpassword para crear el fichero de 
contraseñas. El módulo de autenticación necesario suele venir con Apache 
pero no habilitado por defecto. Para activarlo usamos la utilidad a2enmod y, 
a continuación reiniciamos el servidor Apache:

```
$ a2enmod auth_digest
$ /etc/init.d/apache2 restart
```

Luego incluimos una sección como esta en el fichero de configuración de 
nuestro Virtual Host:

```
  <Directory "/var/www/miweb/privado">
     AuthType Digest
     AuthName "dominio"
     AuthUserFile "/etc/claves/digest.txt"
     Require valid-user
  </Directory>
```

Como vemos, es muy similar a la configuración necesaria en la autenticación 
básica. La directiva AuthName que en la autenticación básica se usaba para 
mostrar un mensaje en la ventana que pide el usuario y contraseña, ahora se 
usa también para identificar un nombre de dominio (realm) que debe de 
coincidir con el que aparezca después en el fichero de contraseñas. Dicho esto, 
vamos a generar dicho fichero con la utilidad htdigest:

```
$ htdigest -c /etc/claves/digest.txt dominio josemaria
Adding password for josemaria in realm dominio.
New password:
Re-type new password:
```

Al igual que ocurría con htpassword, la opción -c (create) sólo debemos de 
usarla al crear el fichero con el primer usuario. Luego añadiremos los 
restantes usuarios prescindiendo de ella. A continuación vemos el fichero que 
se genera después de añadir un segundo usuario:

```
josemaria:dominio:8d6af4e11e38ee8b51bb775895e11e0f
gemma:dominio:dbd98f4294e2a49f62a486ec070b9b8c
```

**Cómo funciona este método de autentificación**

Cuando desde el cliente intentamos acceder a una URL que esta controlada por 
el método de autentificación de tipo digest:

1. El servidor manda una respuesta del tipo 401 HTTP/1.1 401 Authorization 
Required con una cabecera WWW-Authenticate al cliente de la forma:

```
     WWW-Authenticate: Digest realm="dominio", 
                      nonce="cIIDldTpBAA=9b0ce6b8eff03f5ef8b59da45a1ddfca0bc0c485", 
                      algorithm=MD5, 
                      qop="auth"
```

2. El navegador del cliente muestra una ventana emergente preguntando por el 
nombre de usuario y contraseña y cuando se rellena se manda una petición con 
una cabecera Authorization

```
     Authorization	Digest username="jose", 
                     realm="dominio", 
                     nonce="cIIDldTpBAA=9b0ce6b8eff03f5ef8b59da45a1ddfca0bc0c485",
                     uri="/digest/", 
                     algorithm=MD5, 
                     response="814bc0d6644fa1202650e2c404460a21", 
                     qop=auth, 
                     nc=00000001, 
                     cnonce="3da69c14300e446b"
```

La información que se manda es responde que en este caso esta cifrada usando 
md5 y que se calcula de la siguiente manera:

* Se calcula el md5 del nombre de usuario, del dominio (realm) y la 
contraseña, la llamamos HA1.

* Se calcula el md5 del método de la petición (por ejemplo GET) y de la uri a 
la que estamos accediendo, la llamamos HA2.

* El reultado que se manda es el md5 de HA1, un número aleatorio (nonce), el 
contador de peticiones (nc), el qop y el HA2.

Una vez que lo recibe el servidor, puede hacer la misma operación y comprobar 
si la información que se ha enviado es válida, con lo que se permitiría 
el acceso.

Crea un escenario en Vagrant o reutiliza uno de los que tienes en ejercicios 
anteriores, que tenga un servidor con una red publica, y una privada y un 
cliente conectada a la red privada. Crea un host virtual 
_departamentos.iesgn.org_.

    
**Tarea 1.** A la URL departamentos.iesgn.org/intranet sólo se debe tener 
acceso desde el cliente de la red local, y no se pueda acceder desde la 
anfitriona por la red pública. A la URL departamentos.iesgn.org/internet, 
sin embargo, sólo se debe tener acceso desde la anfitriona por la red pública, 
y no desde la red local.

En la configuración del VirtualHost _/etc/apache2/sites-avalaible/departamentos.conf_
añadimos las siguientes líneas:

```
<Directory /var/www/departamentos/internet/>
                Options Indexes FollowSymLinks MultiViews
                <RequireAll>
                        Require not ip 192.168.100
                        Require all granted
                </RequireAll>
</Directory>
```

Esto hará que solo sea posible acceder a dicho directorio a través de una
dirección IP que esté fuera de nuestra red local.

Y también, añadiremos lo siguiente:

```
<Directory /var/www/departamentos/intranet/>
                Options Indexes FollowSymLinks MultiViews
                <RequireAll>
                        Require ip 192.168.100
                        Require all granted
                </RequireAll>
</Directory>
```

Esta configuración permite solo a las Ips de la red local entrar a dicho
directorio.

Acto seguido, nos dirigiremos tanto al cliente como a la máquina anfitriona
y modificaremos el fichero /etc/hosts_, añadiendo:

**En el cliente:**

```192.168.100.4   www.departamentos.iesgn.org```

**En la máquina:**

```192.168.0.101   www.departamentos.iesgn.org```

Ahora para comprobar que funciona, vamos a dirigirnos a nuestra máquina e
intentaremos acceder a _intranet_:

![alt text](../Imágenes/compintranet.png)

Pero sin embargo, podemos acceder a _internet_:

![alt text](../Imágenes/compinternet.png)

Y ahora comprobaremos en el cliente:

![alt text](../Imágenes/compintranet2.png)

![alt text](../Imágenes/compinternet2.png)

**Tarea 2.** Autentificación básica. Limita el acceso a la URL 
_departamentos.iesgn.org/secreto__. Comprueba las cabeceras de los mensajes 
HTTP que se intercambian entre el servidor y el cliente. ¿Cómo se manda la 
contraseña entre el cliente y el servidor?. Entrega una breve explicación del 
ejercicio.

Crearemos el directorio secreto en primer lugar. Después crearemos en 
_/etc/apache2_ un directorio llamado login por ejemplo y una vez allí,
ejecutamos la siguiente instrucción:

```
htpasswd -c /etc/apache2/login/pass.txt root
```

Así crearemos las credenciales para root. Y ahora modificaremos de nuevo la 
configuración del VirtualHost:

```
<Directory /var/www/departamentos/secreto> 
                Options Indexes FollowSymLinks MultiViews
                AuthType Basic
                AuthName "Introduzca sus datos"
                AuthUserFile "/etc/apache2/login/pass.txt"
                Require valid-user
</Directory>
```

Reiniciamos apache2 y desde nuestra máquina física, accedemos al directorio
_secreto_:

![alt text](../Imágenes/validuser.png)

Introducimos los datos y accedemos sin problema:

![alt text](../Imágenes/validuser2.png)

Y desde el cliente, haremos lo siguiente:

```
vagrant@Cliente:~$ curl -i 'www.departamentos.iesgn.org/secreto'
HTTP/1.1 401 Unauthorized
Date: Tue, 27 Oct 2020 19:24:57 GMT
Server: Apache/2.4.38 (Debian)
WWW-Authenticate: Basic realm="Introduzca sus datos"
Content-Length: 474
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>401 Unauthorized</title>
</head><body>
<h1>Unauthorized</h1>
<p>This server could not verify that you
are authorized to access the document
requested.  Either you supplied the wrong
credentials (e.g., bad password), or your
browser doesn't understand how to supply
the credentials required.</p>
<hr>
<address>Apache/2.4.38 (Debian) Server at www.departamentos.iesgn.org Port 80</address>
</body></html>
vagrant@Cliente:~$ 
```

La contraseña entre el servidor y el cliente se envía encriptada en Base64.

**Tarea 3.** Cómo hemos visto la autentificación básica no es segura, modifica 
la autentificación para que sea del tipo digest, y sólo sea accesible a los 
usuarios pertenecientes al grupo directivos. Comprueba las cabeceras de los 
mensajes HTTP que se intercambian entre el servidor y el cliente. ¿Cómo 
funciona esta autentificación?

Primero, activaremos el módulo:

```
sudo a2enmod auth_digest
```

Y modificamos lo añadido en el anterior ejercicio en la configuración del
VirtualHost:

```
<Directory /var/www/departamentos/secreto>
                Options Indexes FollowSymLinks MultiViews
                AuthType Digest
                AuthName "directivos"
                AuthUserFile "/etc/apache2/login/pass.txt"
                Require valid-user
</Directory>
```

Y añadiremos dos usuarios, uno que tenga acceso y otro que no:

```
vagrant@Servidor:/etc/apache2/sites-available$ sudo htdigest -c /etc/apache2/login/pass.txt directivos lora
Adding password for lora in realm directivos.
New password: 
Re-type new password: 
vagrant@Servidor:/etc/apache2/sites-available$ sudo htdigest -c /etc/apache2/login/pass.txt sinacceso manuel
Adding password for manuel in realm sinacceso.
New password: 
Re-type new password: 
```

Y reiniciamos el servicio apache otra vez y comprobamos que lora tiene acceso
mientras que manuel no.

![alt text](../Imágenes/lorasi.png)

![alt text](../Imágenes/lorasi2.png)

![alt text](../Imágenes/manuelno.png)

![alt text](../Imágenes/manuelno2.png)

**En el cliente:**

```
vagrant@Cliente:~$ curl -i 'www.departamentos.iesgn.org/secreto'
HTTP/1.1 401 Unauthorized
Date: Tue, 27 Oct 2020 19:43:58 GMT
Server: Apache/2.4.38 (Debian)
WWW-Authenticate: Digest realm="directivos", nonce="useYRKyyBQA=92998a743c2095b7718b7e55196aaab8db70f099", algorithm=MD5, qop="auth"
Content-Length: 474
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>401 Unauthorized</title>
</head><body>
<h1>Unauthorized</h1>
<p>This server could not verify that you
are authorized to access the document
requested.  Either you supplied the wrong
credentials (e.g., bad password), or your
browser doesn't understand how to supply
the credentials required.</p>
<hr>
<address>Apache/2.4.38 (Debian) Server at www.departamentos.iesgn.org Port 80</address>
</body></html>
```

Con Digest ahora las credenciales están cifradas en MD5.

**Tarea 4.** Vamos a combinar el control de acceso (tarea 6) y la 
autentificación (tareas 7 y 8), y vamos a configurar el virtual host para que 
se comporte de la siguiente manera: el acceso a la URL 
_departamentos.iesgn.org/secreto_ se hace forma directa desde la intranet, 
desde la red pública te pide la autentificación. 
Muestra el resultado al profesor.

Para ello, cambiaremos la configuración de _/secreto/_ a la siguiente:

```
<Directory /var/www/departamentos/secreto>
                Options Indexes FollowSymLinks MultiViews
                AuthType Digest
                AuthName "directivos"
                AuthUserFile "/etc/apache2/login/pass.txt"
                <RequireAny>
                        Require ip 192.168.100
			Require valid-user
                </RequireAny>
</Directory>
```

Y reiniciamos apache2 otra vez. Ahora con las comprobaciones:

Primero vamos a comprobar que desde nuestra máquina física, nos pedirá
autentificación:

![alt text](../Imágenes/digestip.png)

Y ahora en el cliente, no nos pedirá confirmación ninguna:

![alt text](../Imágenes/digestip2.png)
