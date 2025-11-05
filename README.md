
<img  align="left" width="150" style="float: left;" src="https://www.upm.es/sfs/Rectorado/Gabinete%20del%20Rector/Logos/UPM/CEI/LOGOTIPO%20leyenda%20color%20JPG%20p.png">
<img  align="right" width="60" style="float: right;" src="http://www.dit.upm.es/figures/logos/ditupm-big.gif">


<br/><br/>


# PRÁCTICA 3: GESTIÓN DE SERVICIOS

## Objetivos
- Gestión e instalación de aplicaciones y servicios
- Configuración de servidores
- Instalación en una nube pública
## Actividades a desarrollar
El objetivo de esta práctica es aprender a instalar y configurar un servicio, el servidor web NGINX en este caso tanto en una máquina virtual como en la nube. Como todos ellos se basan en máquinas virtuales lo realizaremos sobre estas. Trabajando para nuestra empresa se ha decidido que el alojamiento de páginas web (web hosting) es una nueva línea prometedora de negocios. Para ello vamos a automatizar los procedimientos necesarios para automatizar la creación de nuevos sitios web a nuestros clientes. Para ello usaremos una infraestructura de máquinas virtuales donde se instalarán servidores HTTP que servirán las páginas de nuestros clientes. Se ha seleccionado el servidor NGINX por ser uno de los más ampliamente usados para dicha labor. Se automatizará mediante uno o más scripts usando Python.
## Instalación y configuración manual de los servicios.
- Los pasos a realizar son:
- Crear una máquina virtual.
- Ver los pasos de instalación.
- Construir un script que los automatice.



## Implementación del escenario de máquinas virtuales
Para el arranque usar la práctica anterior.  Se va a implementar la infraestructura de la empresa usando una máquina física donde se podrán ejecutar máquinas virtuales que contendrán los servidores.

Para poder desarrollar la práctica, se debe disponer de dos máquinas: la máquina que ejecutará el proceso servidor, y la máquina que actuará como cliente. 

Con el objetivo de reducir el peso del escenario, y facilitar la ejecución de la práctica, sólo la máquina que ejecute el servidor va a ser de tipo virtual. 
Como máquina cliente se empleará la máquina anfitriona (el propio puesto del laboratorio).

![escenario.jpg](img/scenario.jpg)


La configuración del escenario puede verse en la imagen anterior. Para conseguir arrancar dicho escenario, la configuración de la biblioteca libvirt de los puestos del laboratorio se ha modificado para que el rango de direcciones de red entre la IP 192.168.122.201 y la IP 192.168.122.254 no sea manejado por el servidor DHCP de la biblioteca, sino que quede libre para asignar las direcciones de forma estática.
Se debe, entonces, arrancar una máquina virtual en la que se pueda disponer de permisos de superusuario, que hará las veces de servidor, y a la que debemos asignarle de forma estática la dirección de red indicada en la figura. Para ello, en primer lugar, se debe seguir el mismo proceso que se empleó en el primer apartado de la práctica 1. 

## Instalación de los servicios

Una vez hemos arrancado el escenario, y hemos comprobado que existe conectividad entre ambas máquinas se procede a instalar un servidor NGINX en la máquina virtual. Para poder verificarlo desde la consola de comandos usaremos un navegador textual LYNX2. A pesar de su limitada funcionalidad es de gran utilidad. Asimismo, instalaremos una herramienta de descarga de webs que nos permitirá verificar si una página web existe WGET (aunque también se puede utilizar CURL)

```bash
sudo apt-get update
sudo apt-get install lynx
sudo apt-get install wget
sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring

curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg]  http://nginx.org/packages/ubuntu `lsb_release -cs` nginx"  | sudo tee /etc/apt/sources.list.d/nginx.list
sudo apt update
sudo apt install nginx
sudo nginx
```

Para la realización posterior de los scripts es necesario recordar que la ejecución de los comandos unix devuelven el resultado de la operación (si ha concluido con éxito o no). Esto se podrá utilizar como condición dentro de un script. Una vez instaladas las herramientas, podemos comprobar que el servidor NGINX está listo mediante:
```bash
lynx http://localhost
```

Una vez se ha comprobado que el servidor está en funcionamiento, es importante familiarizarse con los directorios y ficheros que controlan su funcionamiento. Dos localizaciones en el sistema de ficheros son importantes para el servidor NGINX.

Dependiendo de la instalación, los ficheros se encuentran en /usr/local/nginx/conf, /etc/nginx, o /usr/local/etc/nginx. Busque en que carpeta se encuentran sus ficheros con el comando ls. Por ahora, sólo es de nuestro interés el fichero nginx.conf. Analice su contenido haciendo uso de un editor.
```bash
sudo vi /etc/nginx/nginx.conf
sudo nano /etc/nginx/nginx.conf
```

Arrancar, detener y reiniciar el servidor

Para arrancar, detener o reiniciar el servidor se emplean los siguientes comandos:
```bash
sudo nginx -s SIGNAL
```

Donde SIGNAL puede ser:

- stop : para detener el servidor
- quit : para detener el servidor de forma segura
- reload : para recargar la configuración sin detener el servidor
- reopen : para reabrir los ficheros de log

## Servidor estático

Para crear un servidor web estático, es decir, que sirva páginas HTML simples, basta con crear un directorio donde se almacenen las páginas web y configurar el servidor para que sirva las páginas desde dicho directorio. Para ello, en primer lugar, se crea el directorio donde se almacenarán las páginas web. Crearemos dos carpetas, una para imágenes y otra para las páginas web.
```bash
mkdir /tmp/www
mkdir /tmp/images
```

La documentación para configurar el servidor NGINX puede encontrarse en la web oficial del proyecto https://nginx.org/en/docs/. En concreto, para configurar un servidor web estático, es necesario modificar el fichero de configuración nginx.conf.

Editar el fichero nginx.conf y sustituir el contenido por el siguiente:
```
events {}
http {
    server {
        listen       80;
    }
}
```


Este fichero arranca un servidor web que escucha en el puerto 80. Para servir los ficheros es necesario añadir la localización de los mismos. Añada las siguientes líneas dentro del bloque server:


```cfg
location / {
    root   /temp/www;
}

location /images/ {
    root   /temp;
}
```


A continuación, recargue la configuración del servidor.

En las carpetas creadas anteriormente, cree una página HTML index.html y guarde una imagen en la carpeta de imágenes. Una página web sencilla puede ser:

```html
<html>
    <h1>Servidor funcionando</h1>
    <img src="/images/logo.png" alt="Si ves esto no encuentra el logo">
</html>
```


La cual pide una imagen llamada logo.png. Acceda a la página web desde el navegador Lynx y verifique que la página y la imagen se muestran correctamente.

De forma predeterminada los servidores web buscan un fichero llamado index.html para mostrarlo cuando se accede a una carpeta. Por lo cual al acceder a la raíz del servidor (http://localhost/) se mostrará la página index.html creada anteriormente. En concordancia con aquello es necesario que guarde este fichero como index.html.
Finalmente, compruebe que la página se muestra correctamente accediendo a la URL http://localhost/.

Hasta ahora, sólo hemos comprobado la configuración y funcionamiento del servidor, accediendo a su contenido desde la propia máquina donde se ejecuta. Sin embargo, el objetivo es lograr comunicación desde una máquina que ejecute un proceso cliente de forma remota.
En una primera prueba, se van a emplear las direcciones de red para acceder al contenido. Para ello, basta con iniciar cualquier navegador web en el host y escribir en la barra de direcciones la dirección de red con la que hemos configurado la máquina servidor. Deberemos ver la misma página que nos mostró el navegador lynx. No obstante, conocer la dirección de red de la máquina donde se encuentra el servidor no es común y los usuarios suelen emplear nombres lógicos para acceder a las páginas web (del tipo www.dominio.es). 

Para poder implementar esta funcionalidad en el escenario de la práctica se deben realizar las siguientes configuraciones. Primero, se debe indicar el nombre escogido en el servidor (no es imprescindible, pero sí recomendable). Para ello habría que modificar el archivo de configuración. Añada la siguiente línea dentro del bloque server del fichero nginx.conf:

```cfg
server_name dominio1 www.dominio1.cdps;
```


Tras esto, el servidor ya tiene asociado un nombre lógico. En condiciones normales, para poder acceder al servidor por su nombre lógico desde el cliente (como hemos visto), sería preciso dar de alta una correspondencia en el DNS. Sin embargo, puesto que las direcciones de red que manejamos no son públicas, y no somos dueños de los dominios empleados, no es posible utilizar este servicio. Como alternativa, tenéis que modificar el archivo /etc/hosts, que almacena algunas correspondencias de forma local en el cliente, en todos los puestos del laboratorio. En concreto, se han de añadir las siguientes dos correspondencias:

```cfg
192.168.122.241 dominio1 www.dominio1.cdps
```
Nota: usaremos como dirección IP la que la nube nos asigne a la máquina. Con dicha configuración, introduciendo en el navegador Lynx desde dentro de la máquina la dirección web www.dominio1.cdps, obtendremos como respuesta la página que conocemos desde el servidor NGINX.

En este apartado se va a configurar el servidor NGINX del apartado anterior para que mantenga dos servidores virtuales (ver información complementaria). Para ello, se va tomar como base la configuración del apartado anterior. Primero se deben crear los dos directorios que van a almacenar los contenidos de cada uno de los dominios. Para ello basta ejecutar los comandos:

```bash
mkdir /tmp/dom1
mkdir /tmp/dom2
```

Ahora, dentro de cada directorio, se debe crear una página HTML index.html que identifique a cada uno de los servidores virtuales. Por ejemplo, añada a cada página el nombre del servidor que la sirve: Se repite el proceso para el segundo dominio, pero cambiando su correspondiente index.html para que tenga otro contenido.
Para modificar la configuración del servidor NGINX y añadir el segundo servidor, se añade un segundo bloque server al fichero de configuración nginx.conf. Se debe modificar el server_name y la localización de los ficheros para que apunten al directorio correspondiente.

Finalmente, se debe cargar el nuevo fichero de configuración en el servidor, y activar los cambios: Una vez finalizado este proceso, han quedado habilitados dos servidores virtuales. Para terminar, solo queda comprobar que ambos dominios son accesibles desde el cliente y que el contenido devuelto por el servidor es el apropiado en cada caso.

## Realización de un script que automatice la configuración e instalación de los servicios.
Realizaremos dos scripts de python :

- Install.py : Instalará en una máquina el servidor NGING y realiza la configuración inicial

- Addwebhost.py : Añadirá una nueva web, admitiendo el nombre de dicha web como parámetro.

En un apéndice se incluyen algunos códigos de python para operaciones de manipulación de ficheros para facilitar la realización de la práctica. Nota para copiar ficheros del ordenador de trabajo a la máquina virtual podremos usar el comando SCP. (ejecutar el comando “man scp” para más información)

## Entrega

- Las prácticas de la asignatura se harán por parejas. Deberán entregar un fichero zip que contiene :
- Un fichero pareja.txt con los nombres de los integrantes de
- Capturas de la realización de la configuración de forma manual.
- Ficheros de configuración pedidos.

## Información complementaria python

Llamar a un script de python con parámetros. Ejemplo:
```python
# Usando subprocess
from subprocess import call
call(["ls", "-l"])
# Usando OS command
import os
cmd = 'date'
os.system(cmd)
```


Ejecutar un script de python con parámetros:
```python

import sys
print 'Number of arguments:', len(sys.argv), 'arguments.'
print 'Argument List:', str(sys.argv)
server = sys.argv[2]

Leer un fichero línea a línea:

```python
fin = open("in.txt", 'r') # in file
fout = open("out.txt", 'w') # out file
for line in fin:
    if "cambiar" in line :
        fout.write("nueva linea \n")
    else:
        fout.write(line)
fin.close()
fout.close()
#copiar o mover out.txt a in.txt (si es necesario)
#verificar si funciona con el commando diff in.txt out.txt

```
