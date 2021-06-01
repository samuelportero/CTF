---
description: >-
  Máquina Linux de dificultad fácil en la que aprovecharemos un sistema de
  registro de usuarios poco seguro y a la difusión de información sobre
  contraseñas para acceder al sistema
---

# HTB - Delivery \(easy\)

![](../.gitbook/assets/imagen%20%2814%29.png)

## Escaneo

Lo primero que hacemos es lanzar varios comandos 'nmap' para localizar los puertos abiertos:

![Nmap a los 10000 primero puertos](../.gitbook/assets/imagen%20%2821%29.png)

Sobre estos puertos, tratamos de obtener las versiones de los servicios asociados y posibles vulnerbailidades haciendo uso de scripts de nmap:

![Versiones de los servicios abiertos](../.gitbook/assets/imagen%20%2829%29.png)

Obtenemos más información sobre los servicios:

* OpenSSH 7.9p1 Debian.
* Nginx 1.14.2.
* Mattermost.

## Enumeración

A priori no vemos vulnerabilidades en estos servicios que nos permitan un acceso remoto al sistema, por lo que pasamos a analizar la web en el puerto 80:

![P&#xE1;gina principal](../.gitbook/assets/imagen%20%288%29.png)

Apaceren varias URLs en la página principal:

```text
http://delivery.htb/#contact-us
http://helpdesk.delivery.htb
https://html5up.net (este lo ignoramos de momento, ya que es un recurso de Internet)
```

Para poder acceder a las URL con dominio delivery.htb, editamos el fichero '/etc/hosts' de nuestra máquina para que nos resuelva con la IP del servidor:

```text
10.10.10.222    helpdesk.delivery.htb
10.10.10.222    delivery.htb
```

Ya podemos navegar por la web sin errores de resolución de nombres. En el enlace de contacto, encontramos el siguiente texto:

![P&#xE1;gina de contacto](../.gitbook/assets/imagen%20%281%29.png)

Y un nuevo enlace:

```text
http://delivery.htb:8065/login
```

Accediendo a este enlace, nos aparece una página de login, que además permite la creación de usuarios y reseteo de contraseñas:

![P&#xE1;gina de Mattermost](../.gitbook/assets/imagen%20%2827%29.png)

Nos quedaría por acceder al enlace de 'helpdesk':

![P&#xE1;gina de helpdesk](../.gitbook/assets/imagen%20%2825%29.png)

Navegando por cada una de estas páginas, podemos hacernos una idea de cómo es el flujo de creación de tickets y cuentas de usuario.

## Acceso al sistema

En la página principal nos indican que para poder acceder a Mattermost necesitamos una cuenta '@delivery.htb', que no tenemos. Para conseguirla, nos indican que debemos crear un ticket. 

![Creaci&#xF3;n de ticket en helpdesk](../.gitbook/assets/imagen%20%2824%29.png)

Al crearlo nos dice que podemos actualizar su contenido. bien accediendo a la sección del estado del ticket, o bien enviando un correo a la dirección '&lt;número de ticket&gt;@delivery.htb'. 

![C&#xF3;mo actualizar tickets en helpdesk](../.gitbook/assets/imagen%20%2813%29.png)

¿Qué pasaría por tanto si nos registramos en Mattermost con la dirección de correo que nos facilitaron al crear el ticket? 

![Creaci&#xF3;n de usuario en Mattermost](../.gitbook/assets/imagen%20%2826%29.png)

Pues que el correo de confirmación del usuario aparece directamente en el estado del ticket:

![Enlace de activaci&#xF3;n de correo](../.gitbook/assets/imagen%20%284%29.png)

\(Si no recibimos el correo de activación, probamos a darle a 'recordar contraseña'\):

![Reset de contrase&#xF1;a](../.gitbook/assets/imagen%20%285%29.png)

Accedemos a la URL de confirmación y probamos a acceder a la web de 'Mattermost', encontrándonos con la siguiente información:

![Panel principal de Mattermost tras acceso](../.gitbook/assets/imagen%20%2828%29.png)

Vemos dos cosas interesantes: por un lado lo que parecen ser la contraseña del usuario '**maildeliverer**', que es '**Youve\_G0t:Mail!**', y por otro lado que la contraseña de 'root' podría ser una variante de "**PleaseSubscribe!**". 

Probamos a conectarnos con el usuario 'maildeliverer' y sacamos la flag:

![](../.gitbook/assets/imagen%20%2818%29.png)

{% hint style="success" %}
**USER OWN**
{% endhint %}

## Escalada de privilegios

Del listado de procesos en ejecución, llaman la atención los siguientes:

![](../.gitbook/assets/imagen.png)

![](../.gitbook/assets/imagen%20%2820%29.png)

Accedemos al directorio '/opt/mattermost/' y revisamos todos los subdirectorios y ficheros. En el  fichero de configuración 'config/config.json' encontramos lo que parecen ser las credenciales de la BBDD MariaDB:

![](../.gitbook/assets/imagen%20%2823%29.png)

Probamos a conectarnos con el usuario y contraseña encontrados:

![Acceso a BBDD MariaDB](../.gitbook/assets/imagen%20%2822%29.png)

Una vez conectado a la BBDD, vemos que hay una tabla 'Users'. Probamos a listar todos los registros del usuario 'root':

![Hash del usuario &apos;root&apos; en BBDD](../.gitbook/assets/imagen%20%2810%29.png)

Podemos ver el hash de su contraseña. Como pudimos ver en el panel principal de Mattermost, la contraseña debe ser una variante de 'PleaseSubscribe!'. A esta contraseña le faltaría un número para cumplir con la política, por lo que probamos a añadírselo al final. 

Para realizar el ataque, usamos la herramienta 'Hashcat'. El cifrado que utiliza Mattermost es bcrypt, que se corresponde con el modo 3200 de Hashcat. Además, le indicamos que realice un ataque por fuerza bruta \(-a 3\), le pasamos el fichero con el valor del hash de la contraseña del usuario root, y por último le indicamos que pruebe con todas las posibles variantes de 'PleaseSubscribe!' más un dígito \(?d\):

```text
hashcat -m3200 -a 3 hash.txt 'PleaseSubscribe!?d'
```

![Contrase&#xF1;a no encontrada](../.gitbook/assets/imagen%20%286%29.png)

No encontramos la contraseña, por lo que aumentamos a dos dígitos la final:

```text
hashcat -m3200 -a 3 hash.txt 'PleaseSubscribe!?d?d'
```

En esta ocasión, si encontramos la contraseña, 'PleaseSubscribe!21': 

![Contrase&#xF1;a de root obtenida mediante Hashcat](../.gitbook/assets/imagen%20%287%29.png)

Probamos a conectarnos como 'root' y sacamos la Flag:

![Flag del usuario root](../.gitbook/assets/imagen%20%2819%29.png)

{% hint style="success" %}
**SYSTEM OWN**
{% endhint %}

