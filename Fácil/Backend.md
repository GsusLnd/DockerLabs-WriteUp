# 2. Máquina: Backend 

<a href="https://github.com/GutsNet"><img title="Author" src="https://img.shields.io/badge/Author-GutsNet-purple.svg?style=for-the-badge&logo=github"></a>

**Dificultad:** 🟢 *Fácil*

**Autor:** *[4bytess](https://github.com/4bytess/)*

**Fecha de creación:** *29/08/2024*

**Proceso:**
- [1. Despliegue](#desplegando-la-máquina-vulnerable)
- [2. Escaneo de puertos](#escaneo-de-puertos-con-nmap)
- [3. Sitio web](#visualización-del-sitio-web)
- [4. Escaneo de las bases de datos](#escaneo-de-las-bases-de-datis)
- [5. Escalada de privilegios](#escalada-de-privilegios)

---

## Desplegando la Máquina Vulnerable

```python
~/Backend ᐅ sudo bash auto_deploy.sh backend.tar
```
Este comando ejecuta un script de bash (`auto_deploy.sh`) que despliega la máquina vulnerable a partir del archivo `backend.tar`.

**Salida:**

```python
Estamos desplegando la máquina vulnerable, espere un momento.

Máquina desplegada, su dirección IP es --> 172.17.0.3

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

#### Verificación de Conectividad

```python
~/Backend ᐅ ping -c 2 172.17.0.3
```
El comando `ping -c 2` envía 2 paquetes ICMP a la dirección IP `172.17.0.3` para verificar la conectividad y la respuesta del host.

**Salida:**

```python
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.086 ms
64 bytes from 172.17.0.3: icmp_seq=2 ttl=64 time=0.111 ms

--- 172.17.0.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1004ms
rtt min/avg/max/mdev = 0.086/0.098/0.111/0.012 ms
```

---

## Escaneo de Puertos con Nmap

```python
~/Backend ᐅ nmap -p- -sCV 172.17.0.2
```
Este comando revela que el puerto `80` (HTTP) está abierto ejecutando `Apache` y el puerto `22` con el servicio `SSH`.

**Salida:**

```python
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 08:ba:95:95:10:20:1e:54:19:c3:33:a8:75:dd:f8:4d (ECDSA)
|_  256 1e:22:63:40:c9:b9:c5:6f:c2:09:29:84:6f:e7:0b:76 (ED25519)
80/tcp open  http    Apache httpd 2.4.61 ((Debian))
|_http-server-header: Apache/2.4.61 (Debian)
|_http-title: test page
MAC Address: 02:42:AC:11:00:03 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

**Explicación de parámetros:**

- `-p-`: Escanea todos los puertos (del 1 al 65535).
- `-sCV`: Realiza un escaneo con detección de versión (`-sV`) y utiliza scripts básicos (`-sC`) para obtener más información del servicio.

---
