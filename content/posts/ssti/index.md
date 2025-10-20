---
title: "Resolucion Lab:SSTI - PortSwigger"
date: 2025-10-10
draft: false
summary: "This is my first post on my site"
tags: ["PortSwigger","burpsuite","SSTI"]
---
{{< alert icon="code" cardColor="#1c1c1c">}}
 Este es un laboratorio vulnerable a SSTI, perteneciente a la academia PortSwigger, para mas informacion.


{{< /alert >}}

* https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-with-a-custom-exploit

Para resolver el laboratorio, debemos crear un exploit personalizado para eliminar el archivo `/home/carlos/.ssh/id_rsa`  

Puede iniciar sesión en su propia cuenta utilizando las siguientes credenciales: `wiener`:`peter` 

## Enumeracion de la plantilla
Lo primero que podemos intuir con la informacion que disponemos es que hay un usuario en el sistema llamado `carlos`, SSTI hace referencia a Inyeccion de Plantillas del lado del Servidor.
* Me interesa saber sobre que **motor de plantillas** esta corriendo la aplicacion web, para ello podemos probar lo siguiente

Estos son algunos payloads que podemos probar, fueron extraidos de la web de HACKTRICKS. Dependiendo de la respuesta del servidor podemos tantear el motor de plantillas.

```
{{7*7}} 
${7*7}
#{7*7} 
${7*'7'}
${foobar}
```
Nos logueamos con las credenciales en la aplicacion web y entramos a un post cualquiera, vemos que podemos dejar comentarios al final.
{{< figure 
src="1.png"
caption="Desgraciadamente el motor de plantillas no interpreto ninguna de las entradas"
>}}

Otra accion que podemos realizar es modificar como queremos que se muestre nuestro nombre de usuario.




{{< figure 
src="2.png"
>}}


</br> Le damos a 'Submit' e interceptamos la peticion con burpsuite

{{< figure 
src="3.png"
>}}

Encontramos algo interesante, el parametro es un objeto-valor `user.name` que se le esta pasando directamente a la plantilla.
</br> Modifiquemos el parametro para hacer fallar aproposito la plantilla.
</br> Pasamos solamente el objeto `user` sin ningun atributo
```go
blog-post-author-display=user&csrf=OoafB2qViGSKWGnzy9y0BBFhBQ2YZ6wy
```
Obtenemos entonces lo siguiente

{{< figure 
src="6.png"
>}}

Nos encontramos con el siguiente mensaje de error, del cual nos da informacion sobre la aplicacion, donde al parecer se trata de `Twig` un motor de plantillas para PHP

* En este punto rompimos la aplicacion, lo que debemos hacer es volver a ejecutar el *repeater* de *burpsuite* pero pasandole nuevamente el parametro correcto `user.name`
* Otra alternativa es esperar 20min para que la aplicacion se reinicie, tal como lo indica el laboratorio


{{< figure 
src="7.png"
>}}


## Explotacion de la plantilla(twig)
Otra accion que podemos realizar es cambiar la foto de perfil de nuestro usuario.
</br> Procederemos a subir un archivo de texto como prueba y analizaremos la salida o que devuelve.

```
echo 'prueba' > prueba.txt
```

{{< figure 
src="8.png"
>}}

Encontramos informacion interesante. 
{{< alert icon="bomb" cardColor="#1c1c1c">}}
 El error radica en que no hemos especificado el tipo de archivo, al subir una imagen no es necesario, pero ya que se trata de un archivo de texto debemos especificarlo, por ejemplo `image/png` 

{{< /alert >}}
</br> Hay un archivo `avatar_upload.php` en el directorio home de carlos</br> Donde al parecer hay un metodo `setAvatar('','')` que pertenece a la clase User.php
* El metodo recibe (2) parametros: 
* `1.` El arhivo 
* `2.` El tipo de archivo


### Volvemos a modificar el username
En la peticion que interceptamos anteriormente, pudimos percatarnos que instancia a un objeto `$user` 
### LFI
Vamos a aprovecharnos de la instancia y ejecutar el metodo setAvatar() para forzar un `LFI`
</br> Pasandole como parametro el archivo `/etc/passwd` y engañando a la aplicacion que es una imagen

{{< figure 
caption="le damos a 'send' y enviamos la peticion"
src="9.png"
>}}
Luego de ello debemos dirigirnos al post que hemos comentado anteriormente y actualizamos la pagina, podemos ver que nuestro avatar cambio.
* Es importante que hayamos hecho un comentario anteriormente, para ver los cambios


{{< figure 
src="11.png"
>}}


Vamos a abrir la imagen en una nueva pestaña e interceptaremos la peticion con burpsuite
* BOOM! 💣 Podemos leer los archivos del sistema
{{< figure 
src="10.png"
>}}


### ¿Porque ocurre esto?
{{< alert icon="bomb" cardColor="#1c1c1c">}}
Twig (el motor de plantillas usado en esta aplicación) no está ejecutando PHP crudo (<?php ... ?>). En su lugar, el motor evalúa expresiones de plantilla dentro de su propio contexto. Cuando se le pasa un objeto (por ejemplo $user) al renderizador, Twig permite que la plantilla acceda a atributos y llame métodos siguiendo sus reglas de resolución (user.prop puede mapear a llamadas a getProp(), isProp(), a propiedades públicas o a __call()).



{{< /alert >}}

## Lectura de User.php
Ahora que podemos leer archivos, vamos a leer el archivo `User.php` que se enceuntra en /home/carlos/User.php. Esta informacion la obtuvimos del anterior mensaje de error.
{{< figure 
src="12.png"
caption="Para ver el contenido nos dirigimos al ultimo post que hemos comentado, hacemos click derecho en nuestra foto de perfil y le damos a 'abrir imagen en una pestaña nueva' e interceptamos la peticion"
>}}

El contenido del archivo es el siguiente
```php
HTTP/2 200 OK

Content-Type: image/unknown

X-Frame-Options: SAMEORIGIN

Content-Length: 1681



<?php

class User {
    public $username;
    public $name;
    public $first_name;
    public $nickname;
    public $user_dir;

    public function __construct($username, $name, $first_name, $nickname) {
        $this->username = $username;
        $this->name = $name;
        $this->first_name = $first_name;
        $this->nickname = $nickname;
        $this->user_dir = "users/" . $this->username;
        $this->avatarLink = $this->user_dir . "/avatar";

        if (!file_exists($this->user_dir)) {
            if (!mkdir($this->user_dir, 0755, true))
            {
                throw new Exception("Could not mkdir users/" . $this->username);
            }
        }
    }

    public function setAvatar($filename, $mimetype) {
        if (strpos($mimetype, "image/") !== 0) {
            throw new Exception("Uploaded file mime type is not an image: " . $mimetype);
        }

        if (is_link($this->avatarLink)) {
            $this->rm($this->avatarLink);
        }

        if (!symlink($filename, $this->avatarLink)) {
            throw new Exception("Failed to write symlink " . $filename . " -> " . $this->avatarLink);
        }
    }

    public function delete() {
        $file = $this->user_dir . "/disabled";
        if (file_put_contents($file, "") === false) {
            throw new Exception("Could not write to " . $file);
        }
    }

    public function gdprDelete() {
        $this->rm(readlink($this->avatarLink));
        $this->rm($this->avatarLink);
        $this->delete();
    }

    private function rm($filename) {
        if (!unlink($filename)) {
            throw new Exception("Could not delete " . $filename);
        }
    }
}

?>
```
## Elimicacion del id_rsa
Dentro del codigo hay una funcion interesante, el metodo `gdprDelete()` que elimina nuestro avatar (foto de perfil).
* Para resolver el laboratorio, la consigna nos pide eliminar el archivo `/home/carlos/.ssh/id_rsa`

{{< figure
src="resolver.png"
>}}

Si bien no podemos eliminar directamente el archivo, podemos usar el archivo `id_rsa` como nuestro avatar, tal como lo hicimos anteriormente.


{{< figure 
src="13.png"
caption=""
>}}

Luego realizamos el mismo procedimiento que hicimos anteriormente para ver nuestra foto de perfil e interceptar la peticion


{{< figure 
src="14.png"
caption="Obtenemos esta graciosa respuesta"
>}}

Finalmente procedemos a eliminar el archivo.
</br> Vamos a utilizar el mismo `Repeater` que venimos usando, ya que es donde podemos invocar los metodos del obtejo `$user`


{{< figure 
src="15.png"
caption=""
>}}


Luego de ello nos dirigimos al post que hemos comentado y actualizamos la pagina.


{{< figure 
src="16.png"
caption=""
>}}

La pagina emitira un mensaje, indicandonos que hemos resuelto el Lab :)


## Creacion del script
Lo siguiente es un script en python, que automatiza el procedimiento.

{{< github repo="alialucas7/PortSwigger---SSTI-exploit-" showThumbnail=false >}}



* En `BASE` debemos colocar la URL del Lab, es decir https://***********.web-security-academy.net
* Es importante que tengamos la imagen `logo.jpg` en la misma carpeta que ejecutemos el script




