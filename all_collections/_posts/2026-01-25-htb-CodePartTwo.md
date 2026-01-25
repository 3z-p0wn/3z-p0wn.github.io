---
layout: post
title: HTB CodePartTwo Writeup
date: 2026-01-25 
categories: [HTB]
---

![info.png](/assets/images/2026-01-25-htb-CodePartTwo/info.png)
**CodePartTwo** es una máquina fácil de la plataforma de **HackTheBox** en la que explotaremos la librería **js2py** para ganar acceso al sistema y después ganaremos ejecución remota de comandos como usuario **root** a través de una vulnerabilidad en el sistema de backups.

# 1. Reconocimiento 

Envío un paquete ICMP con **ping** para verificar que la máquina está activa y determinar el sistema operativo de la misma. (En este caso linux por el **ttl** de 63).
![ping.png](/assets/images/2026-01-25-htb-CodePartTwo/ping.png)

Lanzo un escaneo con **nmap** para averiguar que puertos están abiertos:
![nmap.png](/assets/images/2026-01-25-htb-CodePartTwo/nmap.png)

Al parecer hay un servicio web corriendo, así que hago un ataque de fuerza bruta con **gobuster** para descubrir rutas:
![gobuster.png](/assets/images/2026-01-25-htb-CodePartTwo/gobuster.png)
Con **whatweb** puedo ver las tecnologías que utiliza la web, en este caso no hay información relevante:
![whatweb.png](/assets/images/2026-01-25-htb-CodePartTwo/whatweb.png)

En la página principal me permite crear una cuenta o descargar el código fuente de la aplicación.
![index.png](/assets/images/2026-01-25-htb-CodePartTwo/index.png)

Después de iniciar sesión en la web, aparece una página en la que puedo escribir y ejecutar código **JavaScript**.

**Dashboard**:

![dashboard.png](/assets/images/2026-01-25-htb-CodePartTwo/dashboard.png)

En este punto, decido inspeccionar el código de la aplicación para comprobar si utiliza alguna función de evaluación de código insegura:

La funcionalidad de la página se encuentra en el fichero **app.py**:
![apppy.png](/assets/images/2026-01-25-htb-CodePartTwo/apppy.png)

Al parecer, la librería **js2py** tiene una vulnerabilidad en la función **eval_js**: [CVE-2024-28397](https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape).

# 2. Acceso inicial

Para explotar la vulnerabilidad utilizo este [exploit](https://github.com/3z-p0wn/CVE-2024-28397-exploit).
Primero creo un listener con el comando`nc -nlvp <puerto>` y después ejecuto el exploit:
![exploit.png](/assets/images/2026-01-25-htb-CodePartTwo/exploit.png)

Al recibir la consola, la convierto en **tty**:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash");'
Ctrl+z
stty raw -echo; fg
Intro
export TERM=xterm
stty rows <filas> columns <columnas>
```

# 3. Movimiento lateral

En el archivo **~/app/instance/users.db** hay una base de datos **SQL** con el hash de la contraseña del usuario **marco**:

![database.png](/assets/images/2026-01-25-htb-CodePartTwo/database.png)

Este hash, lo crackeo usando [CrackStation](https://crackstation.net/):
![crackstation.png](/assets/images/2026-01-25-htb-CodePartTwo/crackstation.png)

# 5. Root

Con `sudo -l` veo que puedo ejecutar **npbackup-cli** como root:
![sudo-l.png](/assets/images/2026-01-25-htb-CodePartTwo/sudo-l.png)

**npbackup** es utilizado para crear una copia de seguridad de **/home/app/app** en el directorio **backups**:
![backup.png](/assets/images/2026-01-25-htb-CodePartTwo/backup.png)

En el archivo **npbackup.conf**hay un parámetro que permite ejecutar comandos antes y después de que se lleve a cabo la copia de seguridad.

![backupconf.png](/assets/images/2026-01-25-htb-CodePartTwo/backupconf.png)

En el archivo **npbackup.conf**hay un parámetro que permite ejecutar comandos antes y después de que se lleve a cabo la copia de seguridad.

![backupconf.png](/assets/images/2026-01-25-htb-CodePartTwo/backupconf.png)

Como puedo ejecutar **npbackup-cli** como **root**, puedo crear una copia de seguridad utilizando un archivo de configuración malicioso que cree una copia SUID de **/bin/bash** y así ganar acceso root.

Creo una copia de **npbackup.conf**:

```bash
cp npbackup.conf hacked.conf
```

Modifico el archivo **hacked.conf** de la siguiente manera:
![hacked.png](/assets/images/2026-01-25-htb-CodePartTwo/hacked.png)

Creo el backup:

```bash
sudo npbackup-cli -c hacked.conf -b -f
```

Ahora puedo ejecutar comandos como root:

![root.png](/assets/images/2026-01-25-htb-CodePartTwo/root.png)
















