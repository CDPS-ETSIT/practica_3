<img  align="left" width="150" style="float: left;" src="https://www.upm.es/sfs/Rectorado/Gabinete%20del%20Rector/Logos/UPM/CEI/LOGOTIPO%20leyenda%20color%20JPG%20p.png">
<img  align="right" width="60" style="float: right;" src="http://www.dit.upm.es/figures/logos/ditupm-big.gif">


<br/><br/>


# PRÁCTICA 3: GESTIÓN DE SERVICIOS

## Objetivos
* Gestión e instalación de aplicaciones y servicios.
* Configuración de servidores.
* Instalación en una nube pública o entorno virtualizado.

## Actividades a desarrollar
El objetivo de esta práctica es aprender a instalar y configurar un servicio, en este caso el servidor web NGINX. Trabajando para nuestra empresa, se ha decidido que el alojamiento de páginas web (web hosting) es una nueva línea prometedora de negocios. Para ello, vamos a automatizar los procedimientos necesarios para crear nuevos sitios web para nuestros clientes. 

Usaremos una infraestructura con máquinas virtuales donde se instalarán servidores HTTP. Se ha seleccionado el servidor NGINX por ser uno de los más ampliamente usados. Se automatizará el despliegue mediante uno o más scripts usando Python 3.

Los pasos a realizar son:
1. Crear una máquina virtual (basado en la práctica anterior).
2. Ver los pasos de instalación y configuración manual.
3. Construir scripts de Python que automaticen el proceso.

## Implementación del escenario de máquinas virtuales
Se va a implementar la infraestructura de la empresa usando una máquina física donde se podrán ejecutar máquinas virtuales que contendrán los servidores. Para desarrollar la práctica, se debe disponer de dos máquinas: la máquina que ejecutará el proceso servidor y la máquina que actuará como cliente.

Con el objetivo de reducir el peso del escenario, sólo la máquina que ejecute el servidor va a ser de tipo virtual. Como máquina cliente se empleará la máquina anfitriona (el propio puesto del laboratorio).

La máquina virtual deberá estar conectada al bridge por defecto (`virbr0` o el correspondiente de su hipervisor) para tener conectividad con la máquina anfitriona. 

## Instalación y configuración manual de los servicios

Una vez arrancado el escenario y comprobada la conectividad entre la máquina anfitriona y la virtual, se procede a instalar el servidor HTTP NGINX junto con herramientas de terminal como Lynx (un navegador web en modo texto) y Wget, que permitirán realizar comprobaciones rápidas en local.

### 1. Instalación de paquetes
En la máquina virtual (servidor), ejecute:
```bash
sudo apt update
sudo apt install nginx lynx wget
```

### 2. Gestión del ciclo de vida del servicio (systemd)
Es imperativo asegurar que el puerto 80 está libre. Si existe otro servidor web arrancado previamente (como Apache), NGINX fallará al intentar ocupar el puerto. Detenga cualquier servicio conflictivo:
```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
```

En entornos Linux modernos, la gestión de NGINX se delega en `systemd`. Los comandos fundamentales son:
* `sudo systemctl start nginx`: Arranca el servidor.
* `sudo systemctl stop nginx`: Detiene el servidor.
* `sudo systemctl restart nginx`: Reinicia el proceso completo.
* `sudo systemctl reload nginx`: Recarga los ficheros de configuración en caliente sin interrumpir las conexiones activas.

Compruebe el funcionamiento inicial accediendo al servidor con el navegador textual:
```bash
lynx http://localhost
```

### 3. Estructura de directorios y configuración modular
Para alojar los ficheros de nuestros clientes, estableceremos una estructura de directorios propia. Crearemos un espacio para la web estática y otro para recursos multimedia:
```bash
mkdir -p /tmp/www/images
```

Utilizaremos el modelo modular estándar de Debian/Ubuntu. NGINX lee sus configuraciones desde dos directorios clave:
* `/etc/nginx/sites-available/`: Almacena la configuración de todos los sitios (Virtual Hosts) creados.
* `/etc/nginx/sites-enabled/`: Contiene enlaces simbólicos a los ficheros de `sites-available` que el servidor debe cargar realmente en memoria.

Asegúrese de que su archivo `/etc/nginx/nginx.conf` incluye la siguiente directiva dentro del bloque `http {}`:
```nginx
include /etc/nginx/sites-enabled/*;
```

### 4. Creación del Virtual Host y la web estática
Cree un archivo de configuración para su sitio principal ejecutando `sudo nano /etc/nginx/sites-available/sitio_estatico` e introduzca el siguiente bloque:
```nginx
server {
    listen 80;
    
    location / {
        root /tmp/www;
        index index.html;
    }

    location /images/ {
        root /tmp/www;
    }
}
```

A continuación, cree una página web de prueba simple (`index.html`) en la ruta especificada:
```bash
echo "<html><h1>Servidor estático funcionando</h1><img src='/images/logo.png' alt='Logo'></html>" > /tmp/www/index.html
```

### 5. Activación y recarga
Para que NGINX comience a servir esta nueva configuración, elimine la página por defecto y cree el enlace simbólico:
```bash
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/sitio_estatico /etc/nginx/sites-enabled/
```

Valide la sintaxis y aplique los cambios:
```bash
sudo nginx -t
sudo systemctl reload nginx
```
Vuelva a ejecutar `lynx http://localhost` para verificar que la respuesta coincide con su archivo `index.html` personalizado.

## Resolución de Nombres Lógicos (Simulación de DNS local)

Para que NGINX sepa qué página web devolver cuando hay múltiples Virtual Hosts configurados, lee la petición y la compara con la directiva `server_name`. Para que esto funcione, el cliente debe traducir el nombre (ej. `www.dominio1.cdps`) a la IP del servidor.

Al no disponer de un servidor DNS real para dominios de prueba, modificaremos el archivo de resolución local `/etc/hosts` dentro de la propia **máquina virtual**:

1. Edite el archivo:
   ```bash
   sudo nano /etc/hosts
   ```
2. Añada la siguiente línea al final del documento (sustituya la IP por la de su máquina virtual o use `127.0.0.1` para pruebas locales estrictas):
   ```text
   192.168.122.241 dominio1 www.dominio1.cdps
   ```
3. Compruebe la resolución invocando al navegador textual Lynx:
   ```bash
   lynx http://www.dominio1.cdps
   ```

## Realización de un script que automatice la instalación

Realizaremos dos scripts de Python 3:
* `Install.py`: Instalará en la máquina el servidor NGINX y realizará la configuración inicial base.
* `Addwebhost.py`: Añadirá una nueva web de forma modular, admitiendo el nombre del dominio como parámetro.

### Esqueleto base para `Addwebhost.py`
Puede utilizar el siguiente esqueleto aplicando buenas prácticas para iniciar el desarrollo:

```python
#!/usr/bin/env python3
import argparse
import os
import subprocess
import sys

def create_webhost(domain):
    root_dir = f"/tmp/{domain}"
    sites_available = f"/etc/nginx/sites-available/{domain}"
    sites_enabled = f"/etc/nginx/sites-enabled/{domain}"

    print(f"[*] Iniciando configuración del Virtual Host: {domain}")

    # 1. Crear directorio raíz
    # TODO: Utiliza os.makedirs(..., exist_ok=True)
    
    # 2. Crear archivo index.html
    # TODO: Usa el context manager 'with open(...)'

    # 3. Definir y escribir la configuración de NGINX
    nginx_conf = f"""server {{
    listen 80;
    server_name {domain} www.{domain};
    
    location / {{
        root {root_dir};
        index index.html;
    }}
}}
"""
    # TODO: Escribe nginx_conf en 'sites_available' usando 'with open(...)'

    # 4. Habilitar el sitio (symlink)
    try:
        # TODO: Ejecutar 'ln -s' con subprocess.run([...], check=True)
        pass
    except subprocess.CalledProcessError as e:
        print(f"[!] Error al crear el enlace simbólico: {e}")
        sys.exit(1)

    # 5. Comprobar sintaxis y recargar NGINX
    try:
        # TODO: Ejecutar 'nginx -t' y 'systemctl reload nginx' con subprocess.run
        print(f"[+] ¡Éxito! El dominio {domain} ha sido desplegado.")
    except subprocess.CalledProcessError as e:
        print(f"[!] Fallo durante la configuración. Error: {e}")
        sys.exit(1)

def main():
    parser = argparse.ArgumentParser(description="Automatización de Virtual Hosts en NGINX.")
    parser.add_argument("domain", help="Nombre del dominio a configurar (ej. dominio2.cdps)")
    args = parser.parse_args()

    if os.geteuid() != 0:
        print("[!] Permisos insuficientes. Ejecuta el script con sudo.")
        sys.exit(1)

    create_webhost(args.domain)

if __name__ == "__main__":
    main()
```

## Entrega y Criterios de Evaluación

Las prácticas se realizarán por parejas. Deberán entregar a través de la plataforma un único fichero comprimido (ZIP) que contenga:
* **`grupo.txt`**: Nombre, apellidos y correo de los integrantes.
* **`Install.py`**: Script de Python 3 para el despliegue inicial.
* **`Addwebhost.py`**: Script de Python 3 para el despliegue de Virtual Hosts.
* **Ficheros `.conf` o plantillas** adicionales necesarias para los scripts.

**Criterios de Evaluación:**
1. **Ejecución Limpia:** Los scripts deben ejecutar sin intervención humana en una máquina Ubuntu limpia y los dominios deben responder correctamente.
2. **Arquitectura de Servicios:** Se exige generar archivos en `sites-available` y habilitarlos en `sites-enabled`. Se penalizará modificar directamente `nginx.conf`.
3. **Calidad del Código (Python 3):** Uso obligatorio de `subprocess.run()`, validación de argumentos y *context managers* (`with open()`).
4. **Tolerancia a Fallos:** Controlar que el servicio se recargue solo si la validación previa de sintaxis es correcta.

---

## Anexo: Información complementaria Python 3

**1. Ejecutar comandos del sistema operativo (`subprocess`)**
Se desaconseja el uso de `os.system()`. La forma recomendada es `subprocess.run`:
```python
import subprocess
# Ejecutar un comando simple
subprocess.run(["sudo", "systemctl", "reload", "nginx"])

# Ejecutar y capturar salida
resultado = subprocess.run(["dpkg", "-l", "nginx"], capture_output=True, text=True)
if resultado.returncode == 0:
    print("Paquete instalado.")
```

**2. Paso de argumentos por línea de comandos**
Para recibir parámetros por terminal, acceda a `sys.argv` o utilice `argparse`:
```python
import sys
if len(sys.argv) != 2:
    print(f"Uso correcto: python3 {sys.argv[0]} <nombre_dominio>")
    sys.exit(1)
nuevo_dominio = sys.argv[1]
```

**3. Manipulación de ficheros (Context Managers)**
Utilice el bloque `with` para asegurar el cierre automático de descriptores de ficheros, evitando bloqueos:
```python
with open("in.txt", 'r') as fin, open("out.txt", 'w') as fout:
    for linea in fin:
        if "cambiar" in linea:
            fout.write("nueva linea configurada\n")
        else:
            fout.write(linea)
# No es necesario usar .close()
```
