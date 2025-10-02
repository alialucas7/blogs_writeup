---
title: "Creacion y Explotacion de un Certificado ESC1 en AD"
date: 2025-08-21
draft: false
summary: "This is my first post on my site"
tags: ["certipy","ESC1","AD-CS","certify"]
---


## Agregar Caracteristica de AD-CS en Windows Server
* Si dentro del panel de `Administracion del Servidor` tienes lo siguiente
{{< figure

    src="ad cs.PNG"
>}}
Significa que ya tienes instalada la caracteristica y puedes pasar al paso siguiente.

* En caso contrario, te dejo un link de como instalarlo :)
https://learn.microsoft.com/es-es/windows-server/networking/core-network-guide/cncg/server-certs/install-the-certification-authority




## Creacion de la Plantilla Vulnerable
Abrimos la ventana de ejecucion y escribimos `certsrv.msc` esto abrira la entidad de certificacion de Active Directory

{{< figure

    src="ultimo_creo.PNG"
>}}

Click derecho en la Plantilla de Certificados y picamos en Administrar

{{< figure
    src="captura vbox con cuadrito.png"
>}}
Veremos varias plantillas de certificados, Duplicaremos la plantilla `Firma de Codigo`

{{< figure
    src="captura en vbox_sigue (con cuadrito)).png"
>}}
Editamos las propiedades en la seccion general, le vamos a dar un nombre que podamos identificarlo facilmente, el nombre sera `Custom_ESC1`
{{< figure
    src="primera.png"
>}}
Luego nos vamos a la solapa de Nombre del Sujeto y seleccionamos la opcion: proporcionado por el solicitante

{{< figure
    src="encontrado_cuadrito.PNG"
>}}

{{< alert >}}
  **Warning!**  Windows ya nos emitira un cuadro de dialogo, advirtiendonos que la configuracion puede ser un riesgo potencial de seguridad. Cuidadito
{{< /alert >}}
### Modificamos los permisos de la Plantilla
Navegamos a la solapa de seguridad, donde pdemos ver los usuarios y grupos.
</br> En este caso modificaremos los permisos del grupo `Usuarios del dominio` para que cualquier usuario pueda solicitar un certificado.
</br> Para ello picamos en Agregar 

{{< figure
    src="7.PNG"
>}}
Escribimos usuarios del dominio y le damos a aceptar

{{< figure
    src="8.PNG"
>}}
Es este punto ya tendriamos agregado el grupo de usuarios del dominio, luego de ello le damos permiso de `inscripcion` y le damos a aplicar


{{< figure
    src="10.PNG"
>}}


### Definimos Politicas de aplicacion
Ahora nos vamos a la solapa de Extensiones, seleccionamos la directiva de aplicaciones y le damos a modificar

{{< figure
    src="11.PNG"
>}}


Le damos a agregar en el cuadro 
{{< figure
    src="12.PNG"
>}}


Dentro de las politicas debemos seleccionar `Autenticacion del cliente` justamente esta politica es la que esta asociada a la vulnerabilidad `ESC1` si no se controlan los permisos
{{< figure
    src="13.PNG"
>}}


Luego de agregar la politica le damos a Aceptar
{{< figure
    src="14.PNG"
>}}



Verificamos que la politica se haya agregado, visualizando la descripcion de las directivas.
{{< figure
    src="15.PNG"
>}}

### Publicacion de la Plantilla
Ya configurada la plantilla, debemos publicarla en el certificado.
</br> Regresamos a la ventana de la Autoridad de Certificación (certsrv.msc). Hacemos click con el botón derecho en Plantillas de Certificado → Nuevo → Plantilla de Certificado para Emitir.


{{< figure
    src="16.PNG"
>}}

Buscamos la plantilla vulnerable en la lista y selecciónamos, en nuestro caso la creamos como Custom_ESC1.

{{< figure
    src="17.PNG"
>}}


Finalmente de damos a `OK` para publicarla


## ¿Porque la plantilla ESC1 es vulnerable?
En este punto, es crucial comprender por qué esta plantilla es vulnerable. Esto se debe a que comenzamos a tener una configuración incorrecta como:

* Permitir la manipulación del nombre alternativo del sujeto (SAN) → Los atacantes pueden solicitar un certificado como administrator@dc.luc4s 


{{< alert icon="skull-crossbones" cardColor="#51D1F6" iconColor="#1d3557" textColor="#1c1c1c" >}}

 Este problema ocurre en la Administración de Plantillas de Certificado *(certtmpl.msc)*, en la configuración de `Gestión de Solicitudes` de la plantilla. El error radica en que la opción `Incluir en la solicitud` permite a los usuarios especificar cualquier Nombre Alternativo del Sujeto (SAN), lo que permite a los atacantes solicitar certificados para cuentas de administrador, de dominio o de servicio.
{{< /alert>}}
* Hacer accesible a todos los usuarios del dominio → Cualquier usuario del dominio que pertenezca al grupo de usuarios del dominio puede inscribirse.

{{<alert icon="skull-crossbones" cardColor="#51D1F6" iconColor="#1d3557" textColor="#1c1c1c" >}}
 Este problema ocurre al crear o modificar una plantilla de certificado en certsrv.msc o al configurar los "Permisos de inscripción" en Usuarios y equipos de Active Directory (ADUC). El error radica en permitir que el grupo `Usuarios del dominio` se inscriba en la plantilla o en otorgar permisos de "Inscripción" o "Inscripción automática" a todos los miembros del grupo.
{{</alert>}}
* No se necesita aprobación adicional → No se requiere intervención del administrador para emitir un certificado.

{{< alert icon="skull-crossbones" cardColor="#51D1F6" iconColor="#1d3557" textColor="#1c1c1c" >}}
 Este problema ocurre en la MMC de la autoridad de certificación (certsrv.msc), en "Propiedades de la plantilla de certificado". El error permite la emisión de certificados sin aprobación manual, lo que permite a los atacantes solicitar un certificado de administrador sin generar alertas.
{{</alert>}}
Esta configuración hace posible el ataque ESC1, donde un usuario con pocos privilegios puede solicitar un certificado para una cuenta privilegiada, autenticarse usándolo y escalar privilegios.

## Explotacion de la Plantilla con Certify
Antes de atacar, debemos identificar las plantillas de certificado vulnerables. Para ello, utilizaremos la herramienta `certify`
{{< figure
    src="certify.PNG"
>}}

En mi perfil de `linkedin` escribi un documento detallado de como explotar este Laboratorio
* https://www.linkedin.com/posts/alialucas7_escalar-privilegios-en-ad-activity-7378797924853383168-UXrr?utm_source=share&utm_medium=member_desktop&rcm=ACoAACQun-8BxRVo8BKF0p3LwWSGQNB62RxOZ78
</br>Te invito a leerlo y dejarme una interaccion si te gusto.
* Gracias por leer hasta aca
* Happy Hacking:)

